---
name: serpwow-serp-monitoring
description: >-
  Track how a keyword ranks across search engines and locations using the SerpWow Search API,
  including reading pagination correctly and handling parsing incidents without ingesting bad data.
api: SerpWow Search API
base_url: https://api.serpwow.com/live
operations:
  - search
generated: '2026-08-27'
method: generated
source: openapi/serpwow-search-api-openapi.yml, https://docs.trajectdata.com/serpwow/search-api/overview
---

# Monitor search rankings with SerpWow

Use this when you need live search-engine results for a query, optionally localized, across
Google, Bing, Yahoo, Baidu, Yandex, Naver, Amazon or eBay.

## Before you start

- Authentication is a single `api_key` **query-string** parameter. There is no header form, so the
  key appears in URLs — keep it out of logs you share.
- The base URL is `https://api.serpwow.com/live`. There is no version segment; `live` is the
  real-time path.
- Every successfully retrieved page costs one credit. Requests that do not return HTTP 200 are
  not charged.

## Steps

1. **Run the search** — `search` (`GET /search`).
   - Required: `api_key`, `q`.
   - `engine` selects the search engine and defaults to `google`.
   - `location` takes a free-form place string (e.g. `London,England,United Kingdom`). Setting it
     automatically adjusts `google_domain`, `gl` and `hl` so results match what a local user sees.
     Resolve ambiguous place names first with the free Locations API
     (<https://docs.trajectdata.com/serpwow/locations-api/overview>) — it is rate limited to
     120 requests per minute and costs no credits.
   - `device` is `desktop` (default), `tablet` or `mobile` — desktop and mobile SERPs differ, so
     pick deliberately rather than accepting the default.

2. **Page deliberately.**
   - `page` is 1-based. Read the `pagination` object in the response to learn the total pages.
   - `max_page` concatenates several pages into one response — capped at **5** for real-time
     searches. Each page actually returned costs a credit; if fewer pages exist than requested,
     only the pages returned are charged.
   - Some result types use infinite scrolling and expose `next_page_token` instead of `page`.
   - With `max_page` set, each result carries `position`, `page` and `position_overall` — use
     `position_overall` for ranking across the whole window.

3. **Trim the payload before you spend context on it.**
   - `include_fields` / `exclude_fields` accept comma-separated JSON field names in dot notation.
   - `fields` (selective parsing) returns only named top-level objects on Google Web responses.
   - Leave `include_html` at `false` unless you genuinely need raw SERP HTML — it makes responses
     very large. `hide_base64_images=true` drops inline image data.

4. **Handle the failure modes.** Full catalog in `errors/serpwow-error-codes.yml`.
   - `401` — the key is invalid. Do not retry.
   - `400` — bad parameters or an unsupported combination. Fix the request; do not retry unchanged.
   - `402` — out of credits or a payment problem. Stop; retrying will not help.
   - `429` — a rate limit was hit. Back off.
   - `500` — retry after a delay. These are the only errors that appear in the Error Logs API,
     where they are retained for 3 days.
   - `503` — a parsing incident is live for this result type. The body names the type and a
     `retry_after` value. Honour it and check <https://serpwow.statuspage.io/>.

5. **Fail safe during incidents.** Set `skip_on_incident` so the API declines to serve rather than
   returning partial or unstable data. For an unattended pipeline this is preferable to silently
   ingesting a bad parse.

## Reading errors

Errors are **not** RFC 9457. The envelope is:

```json
{ "request_info": { "success": false, "message": "Supplied api_key is not valid" } }
```

Always branch on `request_info.success`, not on the presence of a data key. Every response carries
an `X-Trace-ID` header (exposed via CORS) — quote it to support.

## Cost and budget awareness

There are **no** rate-limit response headers. You cannot read remaining budget from a response
header; call the free Account API (`GET /account?api_key=…`) to read `credits_remaining`,
`credits_limit` and `credits_reset_at`, plus a live `status` array of platform components.
