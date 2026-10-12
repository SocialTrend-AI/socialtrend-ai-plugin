---
name: command-center-schedule
description: Guidance for using the Command Center schedule MCP tools (get_schedule_calendar, get_schedule_insights, get_schedule_filter_options, reschedule_scheduled_item) together correctly. Use whenever the user asks about their content schedule, calendar, posting mix, ghost slots, or wants to move/reschedule a post.
---

# Command Center schedule tools

Four tools, all scoped to the authenticated user's own account:

- `get_schedule_calendar` — the calendar for a view (`month`/`week`/`day`/`list`) anchored on a date, optionally filtered by category/platform/status/niche
- `get_schedule_insights` — mix balance score, platform tiles, ghost slots for a date range
- `get_schedule_filter_options` — valid values for categories/platforms/statuses/niches
- `reschedule_scheduled_item` — move a scheduled item to a new date/time (write)

## Sequencing

1. Call `get_schedule_filter_options` first if you're not sure a filter value (platform, status, niche) is valid before passing it to the other tools. Its `categories` field (`blog`/`newsletter`/`social`/`video`) is the only valid input for `get_schedule_calendar`'s `categories` arg — its `content_types` field is a *different*, unrelated backend taxonomy (e.g. "UGC", "Avatar Video") and is not valid input there.
2. Prefer `get_schedule_calendar` for "what's scheduled" questions and `get_schedule_insights` for "how's my mix/coverage" questions — don't call insights just to list items, its response is intentionally not per-item.
3. Every read response includes a `timezone` field (the account's IANA timezone, or `"UTC"` if unset) — all dates/times in that response are in that timezone. Use it, don't assume UTC or the caller's local time.

## Rescheduling — read the calendar again after every attempt

- Pass `expected_rev` (`revs[category]` from your most recent `get_schedule_calendar` call — a normalized int per category) whenever you have it — this is how the tool catches concurrent edits.
- If the response has `"conflict": true, "reason": "stale_save"` — the schedule changed since your last read. Call `get_schedule_calendar` again to get the current `rev`, then retry.
- If the response has `"requires_confirmation": true, "reason": "cadence_warning"` — the move exceeds the daily posting cadence for that day. **Show the guardrail info to the user and get explicit confirmation before re-calling** with `confirm_override=True`. Never set `confirm_override=True` on the first attempt.
- Past-dated destinations are always rejected (`error_code: "past_dated_target"`) — the backend does not enforce this, the tool does, so don't try to work around it.
- After any successful reschedule, calling `get_schedule_calendar` again to confirm the new state is good practice, especially before a second dependent write.

## `app_name` means different things per category — this trips people up

For `reschedule_scheduled_item`, `app_name` is required for `social` and `video` categories, but:
- **social**: it's the channel name (e.g. `"facebook"`, `"instagram"`)
- **video**: recurring items are represented as *rules*, not per-channel slots — `app_name` must be the rule's `rule_id` (e.g. `"rule_mrt3k8sc_60staa"`), found in the `rule_id` field of the relevant entry from a prior `get_schedule_calendar` call. Passing a channel name here fails with `occurrence_not_found`.

## Moving a manually-created post (not on a recurring slot)

If the item isn't anchored to a recurring `cc_schedule` slot (e.g. a one-off manually-created post), `from_date`/`from_time` alone can't identify it. Pass `workflow_data_id` (and `workflow_id`, required alongside it for every category except video) from the item's fields in a prior `get_schedule_calendar` response instead.

## Errors

Every tool always returns a dict, never raises — check `"status": false` and read `error_code` for a machine-readable reason (e.g. `invalid_date`, `invalid_time`, `past_dated_target`, `stale_save`, `cadence_warning`) rather than parsing the human-readable `error` string.
