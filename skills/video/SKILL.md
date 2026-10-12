---
name: video
description: Guidance for using the video creation MCP tools (create_video, list_recipes, get_generation_status, show_video_preview, search_voices, search_avatars, post_video, request_video_edit, approve_video_edit, reject_video) together correctly. Use whenever the user wants to generate an AI video, check on one, or search for a voice/avatar to use in one.
---

# Video creation tools

Twelve tools, all scoped to the authenticated user's own account. Clone-avatar creation/configuration is **not** exposed by this tool set — if the user wants a video of themselves from their own photos, say plainly that isn't available here yet.

- `generate_video_script` — generates a narration script and a visual prompt; **always call this before create_video**
- `list_recipes` — recipes for one selected connected account (format/avatar/voice/captions bundles); returns at most `limit` recipes (default 25, max 50), with `count` giving the total
- `search_voices` — added voices for that selected account, or an explicit semantic search of the community voice catalog
- `search_avatars` — added avatars for that selected account, or an explicit structured-filter search of the preset avatar catalog (gender/age/pose/etc. -- not free-text)
- `get_video_credit_cost_estimate` — pre-flight credit/subscription check; call it before `create_video`
- `create_video` — starts async video generation, from a recipe or an ad-hoc configuration; on a widget-capable host, mounts a self-updating card immediately (see "Once the video is ready" below)
- `get_generation_status` — polls a generation job; returns the video + thumbnail once ready
- `show_video_preview` — renders a ready video's card (thumbnail, recipe, confirmed posting platforms) inline; call it exactly once, only after the video is ready
- `post_video` — posts a reviewed video to the platforms/accounts already chosen at `create_video` time
- `request_video_edit` — queues a targeted edit plan from a plain-language change request
- `approve_video_edit` — approves (executes, may deduct credits) or cancels a pending edit plan
- `reject_video` — cancels the video outright; takes only `video_data_id`

## Nothing proceeds without the user confirming first

Before choosing any recipe, avatar, or voice, select exactly one connected platform/account. Use `list_connected_accounts` when the account is unknown or ambiguous; use its human-readable `platform` and `account_name`, never a raw connection id. Before calling `create_video`, **all of the following must be explicitly shown to the user and confirmed**: that same account, format (`ugc`/`faceless`), voice, avatar (if needed), video length, the narration, the post caption (copy), and caption settings. If the user explicitly provided any of those (narration, copy, caption settings) and said to use them as-is, use them verbatim — no rewriting.

## Content intake (do this before account/recipe/voice selection)

Before choosing any technical settings, make sure you have the following. Use whatever the user has already provided — only ask for what is genuinely missing, batched into a single message:

1. **Topic** — what is the video about? (required — if already stated, use it; if too vague or under ~10 words, ask for more detail; do not assume or invent specifics)
2. **Style** — professional, casual, educational, cinematic, storytelling, or entertaining? (required — ask only if not stated)
3. **Key points** — specific facts, statistics, messages, or calls-to-action to include? (optional, but ask if not mentioned)
4. **Target audience** — who is this video for? (optional, but ask if not mentioned)
5. **Visual mood/feel** — e.g. "energetic", "warm and approachable", "authoritative", "sleek and modern", "fast-paced"? (optional, but ask if not mentioned — this directly shapes the background video quality for both UGC and faceless formats)

If the user's opening message already covers topic and style, skip straight to account selection without asking a content-intake question at all.

**Vagueness rule:** If the user says "you decide", "assume everything", or gives only a one-line topic with no other context, ask one focused follow-up to confirm at minimum the topic and style before proceeding. Never generate a video from a vague brief — the backend uses `raw_input` to generate both the spoken narration and the visual background, so richer input = better video.

**How to compose `raw_input`:** Embed all collected context explicitly into `raw_input` when calling `create_video`, e.g.:
> `"Topic: mortgage rate fluctuations in the US. Style: professional. Visual mood: authoritative and calm. Key points: 30-year fixed at 6.66%, focus on affordability over timing, refinancing is an option. Target audience: US homebuyers."`

This structured `raw_input` is the primary driver of both narration quality (spoken audio) and background video quality.

## Sequencing

1. **Collect content inputs** (see above) — one batched question covering topic, style, key points, audience, visual mood.
2. Select one connected account, then call `list_recipes` with that account. Refer to recipes by name only — never show or ask for a recipe id.
3. Let the user choose either a recipe from that account or an ad-hoc combination from its added voice library and, when needed, added avatar library. A recipe is recommended but not mandatory.
4. If no recipe exists, tell the user recipes are created in the SocialTrend app (there is no recipe-creation tool here). Then offer the selected account's added avatar/voice fallback and continue if the user chooses it. If the user names a recipe, pass `recipe_name` to `create_video` (never both `recipe_name` and `recipe_id`) — it's resolved to the internal id server-side. On `error_code=recipe_not_found`, re-ask using the listed available names. On `error_code=ambiguous_recipe_name`, show the user each candidate's `avatar_name`/`voice_name` (never its `recipe_id`) and ask them to choose, then retry `create_video` with the chosen candidate's `recipe_id`.
5. Use `search_voices(source="library")` and, if the format needs one, `search_avatars(source="library")` with the same account. Do not use community voices or preset avatars as the default no-recipe fallback; only search those catalogs when the user explicitly requests them.
6. **Call `generate_video_script`** with the topic, style, video_length, and any key_points/target_audience collected above. This produces a narration script and a visual prompt. **Show the returned `narration_script` to the user and ask for confirmation or edits before proceeding.** Save the `video_data_id`, `narration_script`, and `detailed_video_analysis` from the response.

   **Verbatim bypass:** If the user explicitly provides their own narration and says to use it exactly (e.g. "use this script word for word"), skip `generate_video_script` for narration — no AI rewriting, no `video_data_id` or `detailed_video_analysis`. Similarly, if the user provides their own copy or caption settings verbatim, use those as-is instead of drafting them. In all cases — bypass or not — **show all three to the user for a final confirmation before calling `create_video`**: (1) narration, (2) copy (post caption), (3) caption settings. The confirm step is "here's exactly what I'll use — confirm?" not a draft review.

7. Get a credit estimate, then call `create_video` with exactly that one account in `targets`, the confirmed `narration_script` as `narration`, and pass `video_data_id` and `detailed_video_analysis` from step 6 (only if `generate_video_script` was called). Use either `recipe_id`/`recipe_name` (never both) or the ad-hoc fields (`format`, `voice_id`, `avatar_id`, `video_length`, `avatar_ai_ratio`, caption settings) — never mix recipe and ad-hoc fields.
8. Poll `get_generation_status` with the returned `video_data_id` until `display_status` leaves `"generating"`.

## `format`/`avatar_ai_ratio`

`format="faceless"` with `avatar_ai_ratio > 0` **is** the "mixed" case (avatar overlay on AI visuals) — there is no separate backend "mixed" format value. `avatar_id` is required whenever `format="ugc"` OR `avatar_ai_ratio > 0`, and should be omitted for pure faceless (ratio 0).

## Once the video is ready

Every time `get_generation_status` reports `display_status="ready"`, your response must include `video_url` as a plain clickable link, in the same message — MCP cannot embed video playback in this text response, and the link is the only way the user can actually watch it, so it must never be omitted even if the user didn't ask, even after `show_video_preview` has already rendered a card (that card is a player, not a substitute for this link on every client). Show this before doing anything else with the video.

`create_video` now renders a self-updating card immediately on a widget-capable host: calling it mounts the video's card in a "generating" state (blue spinner banner, never the ready state's green checkmark), and the card's own polling switches it to the finished video automatically once ready — no further tool call needed from you. Do **not** loop-poll `get_generation_status` waiting for the card to finish on such a host; just tell the user generation has started and let the card update itself. Only call `get_generation_status`/`show_video_preview` again if the user explicitly asks to see the status or the card again, or the card's own copy indicates it stopped auto-updating (it says something like "ask me to check on it").

`get_generation_status` still never renders a widget itself, and `show_video_preview` is still the one-shot re-render fallback — poll `get_generation_status` as often as needed while generating. Once (and only once) `display_status` reaches `"ready"` (or `"posted"`), call `show_video_preview` with the returned `video_data_id` exactly once to render the thumbnail/recipe/platform-logo card inline — never while still `"generating"`/`"stalled"`, and never more than once per ready video. Once you've called it, don't narrate the rendered card's contents back to the user, but still always show `video_url` as a plain clickable link.

**The video is never auto-posted.** Once the card is ready (via either path), tell the user, once, that they can post it (`post_video`), ask for changes (`request_video_edit`), or reject it (`reject_video`). Only call `post_video` after the user has reviewed the video via the `video_url` link and explicitly confirms posting — platforms/accounts were already chosen during `create_video` and are not re-specified here. For a targeted tweak (not a full regenerate), call `request_video_edit` with a plain-language `change_input`, then poll `get_generation_status` for the resulting edit plan (`pending_edit_plan.plan`). Once populated, show the full plan and the original `requested_change` to the user and ask them to approve or discard it, then call `approve_video_edit` with `action="approved"` or `action="cancelled"` accordingly. Approving actually executes the plan (re-renders the video, may deduct credits); cancelling discards it at no cost. Never call `approve_video_edit` before the user has seen the plan and explicitly chosen. `reject_video` takes only `video_data_id` — never confuse it with the generic `reject_content_item` approval-queue tool, which needs a `w_category_id` you don't have for a video from this flow. Don't repeat any of this guidance on every later status check — only the first time the ready card is shown.

## Errors

Every tool always returns a dict, never raises — check for an `"error"` key and read `error_code` for a machine-readable reason (e.g. `insufficient_credits`, `content_rejected`, `invalid_platforms`, `video_not_found`, `recipe_not_found`, `ambiguous_recipe_name`) rather than parsing the human-readable `message` string. `reject_video`'s `"conflict": true, "reason": "already_processed"` means the video's state changed since you last checked (already rejected/posted); `"error_code": "not_authorized"` means it isn't the caller's video. It's a consequential write — confirm the specific video with the user before calling it.

`generate_video_script` can return `error_code: VIDEO_SUBSCRIPTION_REQUIRED` — if it does, tell the user plainly that an active video subscription is required and stop. Do not proceed to `get_video_credit_cost_estimate` or `create_video`.
