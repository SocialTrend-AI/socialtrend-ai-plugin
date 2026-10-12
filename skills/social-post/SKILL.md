---
name: social-post
description: Guidance for using the social post image MCP tools (create_social_post_image, get_social_post_status, show_social_post_preview, approve_social_post, reject_social_post, request_social_post_edit) together correctly. Use whenever the user wants to create, preview, approve, reject, or request edits to an AI-generated social media post image.
---

# Social post image tools

Six tools relevant to this flow, all scoped to the authenticated user's own account:

- `create_social_post_image` — starts async generation of an AI image + caption from a natural-language request. Also renders a self-updating "generating" card immediately (see below) — the client no longer needs to poll a status tool to see progress.
- `get_social_post_status` — a plain status read; the widget card now polls a backend endpoint on its own, so you rarely need to call this in a loop anymore (see below)
- `show_social_post_preview` — renders a ready post's images, caption, and hashtags inline as a card; call it exactly once, only after the post is ready (fallback path — normally the card from `create_social_post_image` has already switched itself to ready by the time you'd call this)
- `approve_social_post` — approves the post and kicks off publishing immediately; also how a posting-destination platform is assigned. It polls briefly for the real publish outcome before returning (see `publish_status` below) — don't assume success just because the call didn't error.
- `reject_social_post` — rejects the post instead of publishing it. Takes only `workflow_id`/`workflow_data_id` — no other argument needed.
- `request_social_post_edit` — asks for changes instead of approving or rejecting outright. Same `workflow_id`/`workflow_data_id` as above, plus `change_input` describing the requested change.

`reject_social_post`/`request_social_post_edit` are thin social-only wrappers over the generic approval-queue tools `reject_content_item`/`request_content_edit` (used elsewhere for blog/newsletter/video items reached via `list_pending_approvals`) — those generic tools need a `w_category_id` argument that a post from this flow was never given a way to learn, so use the social-specific wrapper here instead of the generic tool.

## Sequencing

1. Ask the user for the desired image format/type (portrait, landscape, square, or an aspect ratio like "4:5") before calling `create_social_post_image` — it drives the generated image's dimensions and is required.
2. Call `create_social_post_image`. Its response renders an inline card that starts in a "generating" state and switches itself to the ready state automatically once the backend job completes — do not poll `get_social_post_status` in a loop waiting for this; only call it again if the user asks, or if the card's own text says it stopped auto-updating (it gives up after ~10 minutes wall-clock). The tool's response text tells you, once, that the user can approve/reject/request edits once the card is ready — pass that along to the user once, don't repeat it on every turn.
3. Once ready, call `approve_social_post` (publish), `reject_social_post` (reject), or `request_social_post_edit` (ask for changes) per the user's decision. There is no separate publish step for approval — approving *is* publishing — but the actual platform call (Facebook/X/etc.) runs asynchronously right after, and can still fail there even though approval itself succeeded. Make sure the user is ready before calling it, since there's no undo once it's queued.

`approve_social_post` waits a few seconds and returns `publish_status`: `"posted"` (tell the user it's live), `"failed"` (`publish_error` is set — tell the user publishing failed and the post is now marked Failed in SocialTrend's Automation view, where it can be retried), or `"pending"` (still processing — tell the user it's queued, not published, and that you'll need to check `get_social_post_status` again shortly to confirm). Never tell the user the post is "published"/"live" unless `publish_status` is `"posted"`.

## Showing the ready preview

The card from `create_social_post_image` normally switches itself to the ready state without any further tool call — `show_social_post_preview` is a fallback for when the client shows no card at all, or the card stopped auto-updating and the user asks to check again. Call it with the same `workflow_id`/`workflow_data_id` at most once per ready post, only after `get_social_post_status` (or the card) confirms `"waiting_for_approval"`. Once you've called it, don't narrate the rendered card's contents back to the user — the fallback text response it returns (with the same URLs/caption/hashtags) is only for a client that shows no card.

## The posting-destination platform is a separate decision from image format — except for carousels

`platforms` in `create_social_post_image` is about *where the post will be published* — Facebook, Instagram, LinkedIn, or X. It is not the same thing as `image_format`, which only controls image dimensions.

- **For a carousel (`post_type="carousel"`), `platforms` is required, not deferrable.** A carousel's slide count is capped by the destination platform's own limit (e.g. X caps at 4, LinkedIn at 20), so the platform must be chosen *before* calling `create_social_post_image` — ask for it explicitly, batched together with the other pre-creation questions (image format, slide count), the same way `video`/SKILL.md requires format/voice/platform to all be confirmed upfront before `create_video`. Calling `create_social_post_image` with `post_type="carousel"` and no `platforms` now fails outright (`error_code: "invalid_arguments"`) — don't call it speculatively and let that error be how you find out to ask; ask first.
- For a single-image post (`post_type` omitted or `"single_image"`), `platforms` is still **optional**: if the user already knows where they want to post, pass it at creation time; if they don't, `create_social_post_image` still has to size the image for some default format (e.g. portrait → Instagram-sized). That sizing is **not** the user's posting choice, even though `get_social_post_status` may show a populated `platforms` value once generation completes.
- Don't present the platform shown on the draft as the user's choice — don't say things like "it's currently set for Instagram, want to keep it?". That implies a choice was already made on their behalf, when it wasn't. (This only applies to the single-image deferred case above — a carousel's platform is always the user's real choice.)
- Before calling `approve_social_post` for a deferred single-image post, ask a plain, neutral question: "Which platform would you like to post this to?" — unless the user already explicitly named one earlier in the conversation.
- `create_social_post_image`'s response includes `platforms_assigned` (`true`/`false`) — `false` means the platform on the record only reflects the image size and must be confirmed with the user before approving. Always `true` for a carousel, since `platforms` is required to create one.

## The hard block — don't rely on it to prompt the user after the fact

If `approve_social_post` is called without `platforms` on a post whose platform was never explicitly assigned, it now fails with `error_code: "platform_not_selected"` instead of silently approving. This is a real safety net, but asking the user *before* calling the tool is still required — don't call it speculatively and let the error surface as the way you find out you needed to ask; that produces a confusing failed call the user didn't expect.

## Reassigning the platform after preview

Pass `platforms` to `approve_social_post` any time the post needs a posting destination — whether it never had one, or the user wants to change it from what was set at creation. There is no separate "assign platform" tool. This does not apply to a carousel, since its platform is already required at creation and normally isn't reassigned; if it is reassigned to a platform whose slide cap is smaller than the carousel's own slide count, the call fails with `error_code: "carousel_slide_count_conflict"` rather than silently dropping slides — tell the user plainly and let them pick a platform that fits or accept fewer slides on the next creation.

## Errors

Every tool always returns a dict, never raises — check for an `"error"` key and read `error_code` for a machine-readable reason (e.g. `content_rejected`, `invalid_state`, `missing_platforms`, `platform_not_selected`, `invalid_slide_count`, `carousel_slide_count_conflict`) rather than parsing the human-readable `message` string.
