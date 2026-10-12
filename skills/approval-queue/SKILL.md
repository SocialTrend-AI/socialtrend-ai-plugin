---
name: approval-queue
description: Guidance for using the approval-queue MCP tools (list_pending_approvals, review_pending_posts, approve_content_item, reject_content_item, request_content_edit) together correctly. Use whenever the user asks about their approval queue, wants to approve/reject/edit pending content, or asks what's waiting for review.
---

# Approval queue tools

Five tools, all scoped to the authenticated user's own account:

- `list_pending_approvals` — everything awaiting approval across blog, newsletter, social, and video; returns a page at a time (`limit` 1-100, default 25) — pass `next_offset` back as `offset` to get the next page
- `review_pending_posts` — the same queue for social/video/newsletter items only, rendered as an interactive review widget (`limit` 1-25, default 10)
- `approve_content_item` — approve an item, optionally moving it to a new date/time in the same call
- `reject_content_item` — reject/cancel an item
- `request_content_edit` — request an AI-driven edit to an item's content (doesn't change its approval status)

## Sequencing

1. Call `list_pending_approvals` first — every write tool needs `workflow_id`/`workflow_data_id`/`w_category_id` from an item in its response.
2. **Every write tool is consequential** — it approves/rejects/requests changes to real content that would otherwise post or send. Always confirm the specific item (title, module) and the exact action with the user before calling `approve_content_item`, `reject_content_item`, or `request_content_edit`. Never chain a write immediately after a read without that confirmation.

## Prefer the review widget when the queue has media

Prefer `review_pending_posts` over `list_pending_approvals` whenever the queue contains social, video, or newsletter items — it renders an interactive widget (image / video / newsletter preview plus Approve, Reject, and Request-edit buttons) instead of a plain list. The widget's own writes reuse this same `approve_content_item`/`reject_content_item`/`request_content_edit` logic unchanged, exposed as `widget_approve_post`/`widget_reject_post`/`widget_request_edit` — those three exist only for the widget UI and are not callable by you directly.

The widget covers social, video, and newsletter only: a queue containing only blog items still needs `list_pending_approvals`, and `review_pending_posts` says so directly when that happens. On a client that shows no card at all, it still degrades to a readable text summary (including each newsletter's `body_url`), so it's never unsafe to call. Actions taken inside the widget are user-driven — once you've called `review_pending_posts`, don't narrate the rendered items back to the user, and don't approve, reject, or edit them yourself on the user's behalf.

## Approve + reschedule in one call

`approve_content_item` can move an item's date/time as part of approving it — pass all four of `from_date`/`from_time`/`to_date`/`to_time` together (all four or none; passing some but not others is rejected with `incomplete_reschedule_args`). Rescheduling a `social`/`video` item also requires `app_name` (same per-category meaning as `reschedule_scheduled_item`), rejected locally with `missing_app_name` if omitted. This reuses the same cadence/conflict handling as the schedule tools' `reschedule_scheduled_item`:
- `requires_confirmation: true, reason: "cadence_warning"` — show the guardrail info and get explicit confirmation before re-calling with `confirm_override=True`.
- `conflict: true, reason: "stale_save"` — re-read the calendar for the current `rev`, then retry.
- Past-dated destinations are always rejected (`error_code: "past_dated_target"`).
- The move is always checked and gated *before* the item is approved, for both auto-pipeline and manual items — a failed move (`stale_save`/`cadence_warning`) never leaves the item approved at its original date/time.

## The manual-mode gotcha

Some newsletter/social items are manually created rather than auto-generated (`generation_mode == "manual"`) and use a different underlying approval path than auto-pipeline items — this is handled transparently by the tools, but the **response shape can look different**:
- A manually-created newsletter that's approved shows `status: "ready"`, not `"approved"` (it moves to a separate send step, unlike auto-pipeline newsletters).
- A manually-created social post with no platform assigned yet requires `platforms` (e.g. `["facebook", "instagram"]`) on `approve_content_item` — without it, you'll get `error_code: "missing_platforms"` or `"platform_not_selected"`.

You don't need to know an item's `generation_mode` in advance — just be ready for either response shape and pass `platforms` if a `missing_platforms`/`platform_not_selected` error comes back.

## Errors and conflicts

Every tool always returns a dict, never raises — check `"status": false`:
- `"conflict": true, "reason": "already_processed"` — the item was already approved/rejected/moved by something else since you last listed it. Call `list_pending_approvals` again.
- `"error_code": "not_authorized"` — the item doesn't belong to the authenticated account.
- `"error_code": "item_not_found"` — the ids don't match any item.
- Any other `error_code` (e.g. `invalid_platforms`, `edit_limit_reached`) is machine-readable — read it rather than parsing the human-readable `error` string.
