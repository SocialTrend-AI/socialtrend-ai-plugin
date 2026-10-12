---
name: knowledge-base
description: Guidance for using the Knowledge Base source MCP tools (add_kb_source, add_kb_text_source, add_kb_url_source, get_kb_scan_status) together correctly. Use whenever the user wants to add a file, pasted text, or a URL to their SocialTrend Knowledge Base, or check its scan/ingest status.
---

# Knowledge Base source tools

Four tools, all scoped to the authenticated user's own account:

- `add_kb_source` — registers an inline file (plain text, or base64 for PDF/DOCX) as a source
- `add_kb_text_source` — registers a pasted text snippet as a source (stored as a synthesized `.txt` file, same lifecycle as `add_kb_source`)
- `add_kb_url_source` — registers a website, YouTube, or Google-Doc URL as a source
- `get_kb_scan_status` — checks a source's scan (file/text) or scrape (URL) status, plus the overall Knowledge Base ingest status

## No crafting pipeline, unlike content-creation tools

Unlike `create_social_post_image`, there is no LLM prompt-crafting step here. The only safety gate is an automatic PII scan run on every file/text source (never on URLs). Adding a source never rebuilds the Knowledge Base by itself — that stays a separate user/in-app action once the source is confirmed clean.

## Files and pasted text: same lifecycle, one size limit

`add_kb_source` and `add_kb_text_source` both go through the identical `sources[]` + PII-scan lifecycle — a pasted snippet is just treated as a synthesized `.txt` file. Both share a ~4 MB decoded size ceiling. An oversized file/text is rejected outright (`error_code: "payload_too_large"`) and the response directs the user to the in-app upload instead — never a partial or truncated ingest.

- Plain-text formats (txt, csv, md, html) may be passed as text directly.
- Binary formats (pdf, docx) must be base64-encoded with `is_base64=true` on `add_kb_source`.

## URLs are a different pipeline entirely

`add_kb_url_source` does **not** PII-scan and does **not** fetch the URL synchronously. Validation at add-time is limited to scheme, supported host, and dedup (`error_code`s: `invalid_url`, `unsupported_url` for social-media hosts, `url_already_added`). The actual scrape happens at the *next* Knowledge Base build — so a URL can sit at `status: "pending"` for a while before you know whether it succeeded (`"scraped"`) or failed (`"failed"`, with `error_message`).

Don't tell the user a URL has been "added and processed" — say it's been queued for the next build, and that `get_kb_scan_status` is how to check whether it actually worked.

## Status polling

Every `add_kb_*` tool returns a source id (`src_`-prefixed for files/text, `url_`-prefixed for URLs). Poll `get_kb_scan_status` with that id:

- For a `src_` id: `pii_status` (`"not_scanned"` → `"clean"` or `"pii_detected"`, occasionally `"stripped"`/`"kept_as_is"`/`"scan_failed"`/`"content_rejected"` -- the last means the scan found a disallowed embedded-instruction pattern; that source can never be built in, re-add different content instead of retrying) plus the overall `ingest_status`.
- For a `url_` id: `status` (`"pending"` → `"scraped"` or `"failed"`) plus the same overall `ingest_status`. There is no `pii_status` for URLs — don't ask for one.

A source clearing its scan/scrape does not mean the Knowledge Base has been rebuilt yet — `ingest_status` is the separate, overall signal for that, and rebuilding stays a user/in-app action this tool set does not trigger.

## Errors

Every tool always returns a dict, never raises — check for an `"error"` key and read `error_code` for a machine-readable reason (e.g. `payload_too_large`, `unsupported_url`, `url_already_added`, `source_not_found`) rather than parsing the human-readable `message` string.
