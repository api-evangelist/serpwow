---
name: serpwow-product-and-shopping-data
description: >-
  Collect Google Shopping listings, product detail and news coverage for competitive and
  merchandising research with SerpWow, including which product endpoints are being retired.
api: SerpWow Search API
base_url: https://api.serpwow.com/live
operations:
  - shopping
  - product
  - news
generated: '2026-08-27'
method: generated
source: openapi/serpwow-search-api-openapi.yml, https://docs.trajectdata.com/serpwow/product-updates
---

# Product, shopping and news data with SerpWow

## Steps

1. **Shopping listings** — `shopping` (`GET /shopping`). Required: `api_key`, `q`. Returns Google
   Shopping results for a product query. Combine with `location` for market-specific pricing.

2. **A specific product** — `product` (`GET /product`). Required: `api_key`, `product_id`.

   > **Read this before you build on it.** The provider has marked the Google Product search and
   > result surfaces, along with Product Specifications, legacy Product Reviews and legacy Online
   > Sellers, as *deprecating 10/31* in its own documentation navigation. The successor is
   > **Google Product Details**
   > (<https://docs.trajectdata.com/serpwow/search-api/searches/google/product-details>), with new
   > Reviews and Online Sellers surfaces alongside it. New work should target the successors.
   > SerpWow publishes no Sunset/Deprecation response header, so nothing warns you at runtime —
   > the date only exists in the docs. See `lifecycle/serpwow-lifecycle.yml`.

3. **News coverage** — `news` (`GET /news`). Required: `api_key`, `q`. Use `location` and the
   engine's language/country handling for regional coverage.

## Output shaping

- `output` selects `json` (default), `html` or `csv`; `csv_fields` picks the CSV columns.
- `include_fields` / `exclude_fields` take dot-notation JSON paths — use them to cut payloads
  before they reach a model's context window.

## Scaling up

For catalogue-sized work, move to Batches rather than looping real-time calls:

- Up to 15,000 Searches per Batch (100 if `include_html=true`), 10,000 Batches per account.
- Schedule monthly, weekly, daily, hourly, or run on demand.
- Result Sets arrive as JSON, JSON Lines or CSV and are downloadable for **14 days** — pull them
  inside that window or they are gone.
- Register a `notification_webhook` to be told when a Result Set is ready. The receiver must
  respond within **5 seconds**; SerpWow retries 5 times with exponential backoff, then emails.
  There is no webhook signature to verify, so treat the callback as a trigger to fetch, not as
  trusted data. See `asyncapi/serpwow-batches-webhooks.yml`.
- Or push Result Sets straight to your own storage with Destinations (S3, GCS, Azure Blob,
  Alibaba OSS, or any S3-compatible store; 50 per account).

## Reversibility

Batches are a real write surface. Every create has an inverse — Delete Batch, Stop Batch,
Delete/Clear Searches, Delete Destination — but **no window is stated and nothing is restorable
once deleted**, and credits already burned by completed Searches are not refundable by any
documented operation. Stop a runaway Batch immediately rather than deleting and rebuilding it.
