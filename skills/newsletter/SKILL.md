---
name: newsletter
description: Guidance for using the newsletter creation MCP tools (create_newsletter, get_newsletter_status, show_newsletter_preview, publish_newsletter, reject_newsletter, request_newsletter_edit) together correctly. Use whenever the user wants to create, check on, finalize, edit, or reject a newsletter draft.
---

# Newsletter creation tools

Six tools, all scoped to the authenticated user's own account:

- `create_newsletter` — starts async generation of a newsletter draft from raw intent; on a widget-capable host, mounts a self-updating card immediately (see "Showing the draft" below)
- `get_newsletter_status` — polls a generation job; returns the draft (topic, content, preview) once ready
- `show_newsletter_preview` — renders a ready draft's body inline as a sandboxed iframe; call it exactly once, only after the draft is ready
- `publish_newsletter` — finalizes the draft into the in-app "Ready" tab — does **not** send it
- `reject_newsletter` — cancels the draft outright; takes only `workflow_id`/`workflow_data_id`
- `request_newsletter_edit` — asks for an AI-driven change to the draft (`change_input`); takes only `workflow_id`/`workflow_data_id` + `change_input`

## Sequencing

1. Ask the user what the newsletter should be about (topic, sections, tone) — pass their raw description to `create_newsletter` as given. Never write a finished topic/body yourself and pass that instead; the server crafts it from raw intent. For a header image: if the user gives a link they already have, pass `image_url` — it must be a SocialTrend S3 URL from their own account or an approved image host, an arbitrary external URL is rejected; if they attach an image in the conversation, pass `image_base64` + `image_filename` instead — an attached image has no fetchable URL of its own here. Never pass both.
2. Poll `get_newsletter_status` with the returned `workflow_id`/`workflow_data_id` until `generation_status` leaves `"pending"`. Safe to resume polling after a reconnect — job state lives in the backend record, not in any session here.
3. Once ready, `preview["content"]` carries `subject`, `newsletter_image`, and `body_url` (a signed link to the full generated HTML). Present the subject and `body_url` to the user in whatever form renders cleanly in the current client — there's no requirement to echo them verbatim in the same message, but describing the draft in prose without giving the user a way to open `body_url` isn't enough. Do not separately surface `newsletter_image` as its own link/image — it's already embedded as the header image inside the `body_url` HTML itself, so showing it again would just be a duplicate link with no new information. Confirm with the user before calling `publish_newsletter`.
4. `generation_status` reaches one of three completed values — `waiting_for_approval`, `ready`, or `posted` — but only `waiting_for_approval` is eligible for `publish_newsletter`; calling it on an already-`ready` or already-`posted` draft returns `error_code: "invalid_state"`. Check the exact value, not just whether it has left `"pending"`.

## Showing the draft

`create_newsletter` now renders a self-updating card immediately on a widget-capable host: calling it mounts the draft's card in a "generating" state (blue spinner banner, never the ready state's green checkmark), and the card's own polling switches it to the finished draft automatically once ready — no further tool call needed from you. Do **not** loop-poll `get_newsletter_status` waiting for the card to finish on such a host; just tell the user generation has started and let the card update itself. Only call `get_newsletter_status`/`show_newsletter_preview` again if the user explicitly asks to see the status or the card again, or the card's own copy indicates it stopped auto-updating (it says something like "ask me to check on it").

`get_newsletter_status` still never renders a widget itself, and `show_newsletter_preview` is still the one-shot re-render fallback for a text-only host or a mis-sequenced/older-client call — poll `get_newsletter_status` as often as needed while generating. Once (and only once) it reports `generation_status` `"waiting_for_approval"` (or later), call `show_newsletter_preview` with the same `workflow_id`/`workflow_data_id` exactly once to render the newsletter body inline as a sandboxed iframe. Never call it while still `"pending"`, and never more than once per ready draft. Once you've called it, don't narrate the rendered card's subject or body back to the user — the fallback subject/topic/`body_url` it returns as plain text is only for a client that shows no card, and step 3's `body_url` requirement applies there.

Once the card is ready (via either path), tell the user, once, that they can finalize it for review (`publish_newsletter` — restate that this does not send it), ask for changes (`request_newsletter_edit`), or reject it (`reject_newsletter`). Both wrappers take only this draft's own `workflow_id`/`workflow_data_id` — never confuse them with the generic `reject_content_item`/`request_content_edit` approval-queue tools, which need a `w_category_id` you don't have for a draft from this flow. Don't repeat this guidance on every later status check — only the first time the ready card is shown.

## The no-send gotcha

`publish_newsletter` only moves the draft to the in-app "Ready" tab for review — it does **not** send the newsletter to any recipients. **No MCP tool sends a newsletter.** If the user asks to "send" or "publish and send," clarify that finalizing (this tool) and sending are different steps, and sending has to happen in the app itself.

## Credits and moderation

`create_newsletter` consumes the same plan usage as an in-app manual creation — no separate MCP charge. A moderation refusal (`error_code: "content_rejected"`) never reaches the backend, so it never consumes usage either. The refusal message is intentionally generic — never ask the pipeline "why" something was rejected or relay a made-up reason; just tell the user to rephrase.

## Errors and conflicts

Every tool always returns a dict, never raises — check `"status": false`:
- `"error_code": "content_rejected"` — moderation refused the intent; ask the user to rephrase.
- `"error_code": "invalid_state"` (from `publish_newsletter`) — the draft hasn't finished generating yet, or was already finalized. Check `get_newsletter_status` again.
- `"error_code": "ambiguous_image_input"` (from `create_newsletter`) — both `image_url` and `image_base64` were passed; use only one.
- `"error_code": "unsupported_image_host"` (from `create_newsletter`) — `image_url` isn't the user's own SocialTrend S3 link or an approved image host. Don't retry with the same URL; ask the user for a SocialTrend-hosted link, or use `image_base64` if they can attach the image instead.
- `"error_code": "filename_required"` / `"unsupported_image_type"` / `"invalid_base64"` / `"payload_too_large"` (from `create_newsletter`) — problems with `image_base64`/`image_filename`; the message says what to fix.
- `reject_newsletter`/`request_newsletter_edit` — `"conflict": true, "reason": "already_processed"` means the draft's state changed since you last checked (already rejected/edited/finalized); `"error_code": "not_authorized"` means it isn't the caller's draft; `"error_code": "item_not_found"` means the ids don't match anything. Both are consequential writes — confirm the specific draft and (for edits) the requested change with the user before calling either.
- Any other `error_code` is machine-readable — read it rather than parsing the human-readable `error` string.
