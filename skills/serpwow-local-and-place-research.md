---
name: serpwow-local-and-place-research
description: >-
  Pull Google Maps places and their reviews for local-market research with the SerpWow Search API,
  using the free Locations API to make the geography exact before spending credits.
api: SerpWow Search API
base_url: https://api.serpwow.com/live
operations:
  - places
  - placeReviews
generated: '2026-08-27'
method: generated
source: openapi/serpwow-search-api-openapi.yml, https://docs.trajectdata.com/serpwow/search-api/searches/google/places
---

# Local and place research with SerpWow

Use this to find businesses in a geography and then read what people say about them.

## Steps

1. **Pin the geography first.** The Locations API
   (<https://docs.trajectdata.com/serpwow/locations-api/overview>) resolves a free-text place to a
   canonical `location` string. It is free of charge and rate limited to 120 requests per minute,
   so resolving before searching costs nothing and stops you paying credits for the wrong city.

2. **Find places** — `places` (`GET /places`).
   - Required: `api_key`, `q`.
   - Pass the resolved `location`. Google Maps results are strongly location-sensitive; an
     unqualified query returns whatever geography the upstream engine infers.

3. **Read reviews for a place** — `placeReviews` (`GET /place_reviews`).
   - Required: `api_key`, `data_id` — the Google place `data_id`, which you take from the
     corresponding entry in the `places` response. There is no name-based lookup; `data_id` is the
     join key between the two operations.

4. **Paginate reviews.** Reviews are long-tail. Use `page`, and `max_page` (max 5 for real-time)
   to concatenate. Each retrieved page is one credit. Use `position_overall` to order results
   across the whole window.

## Guardrails

- Both operations are read-only. Nothing you do here can be un-done because nothing is written —
  see `conventions/serpwow-conventions.yml`.
- For hundreds or thousands of places, do not loop real-time calls. Use the Batches API
  (<https://docs.trajectdata.com/serpwow/batches-api/overview>): up to 15,000 searches per Batch,
  scheduled or on-demand, with the Batches API itself free of charge — only the Searches it runs
  are charged.
- Watch for `503`. A live parsing incident on Google Places results will produce it; set
  `skip_on_incident` so a nightly job fails cleanly instead of writing a bad snapshot.
