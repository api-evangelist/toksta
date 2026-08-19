---
name: Discover and shortlist B2B creators
description: >-
  Search Toksta's vetted B2B creator database, verify each candidate against real post
  evidence, and save the survivors into a Toksta creator list.
api: openapi/toksta-public-api-openapi.yml
base_url: https://api.toksta.com
operations:
  - searchCreators
  - getSearchResultPosts
  - getCreator
  - createCreatorList
  - saveCreatorsToList
generated: '2026-08-13'
method: generated
source: >-
  Grounded in operationIds verified verbatim in openapi/toksta-public-api-openapi.yml;
  rules from conventions/toksta-conventions.yml and errors/toksta-problem-types.yml.
---

# Discover and shortlist B2B creators

## Before you start

- `Authorization: Bearer tk_live_<secret>` and `Content-Type: application/json` on every call.
- `saveCreatorsToList` and `createCreatorList` are **workspace** endpoints. A dedicated
  API-plan key gets `403 FORBIDDEN` on them; you need a SaaS plan with
  `workspace_endpoints_enabled`. Steps 1-3 work on either plan.
- `searchCreators` costs **1 credit** per call. The evidence and detail reads cost 0.

## Steps

### 1. Search the database — `searchCreators`

`POST /v1/creators/search`

Send `platform` (`linkedin` or `youtube`, defaults to `linkedin`), a free-text `query`
matched against name and bio, and a `filters` object. Useful filters:
`content_categories`, `target_audiences`, `languages`, `locations`, `hide_brands`, and
min/max bands for `followers`, `engagement`, `posts_per_month`, `reactions_per_post`,
`comments_per_post`, `shares_per_post`.

Page with `cursor` + `limit` (1-100, default 50). Sort with `sort_by`
(`followers_count`, `engagement_rate`, `avg_posts_per_month`, `avg_views_per_post`,
`shares_per_post`, `comments_per_post`, `reactions_per_post`, `name`, `updated_at`) and
`sort_order`.

Read `data.creators[]`. Each carries `id`, `name`, `bio`, `platform`, `handle`,
`profile_url`, `location`, `content_categories`, `target_audiences`, `languages`,
`metrics` and `updated_at`.

To paginate, pass the previous response's `meta.next_cursor` back as `cursor`. **Each
page is another billed search call** — set `limit` high rather than paging in small steps.

### 2. Pull the receipts — `getSearchResultPosts`

`POST /v1/search-results/posts`

Send `creator_ids` (the UUIDs from step 1), the `keywords` you care about, and
`posts_limit_per_creator`. This returns the actual posts behind the match, so you can
reject creators whose relevance is incidental. Costs 0 credits — always do this before
spending analysis credits.

### 3. Confirm a single candidate — `getCreator`

`GET /v1/creators/{id}`

Returns the full profile including `created_at`, full `metrics`, and an `enrichment`
object with `is_enriched`. Check `enrichment.is_enriched` here: if a creator is already
enriched, a later enrichment call will not re-charge you.

### 4. Create the shortlist — `createCreatorList`

`POST /v1/creator-lists` with `name`, `platform` and `is_private`. Returns the list id.

Skip this and reuse an existing list id from `listCreatorLists` when you have one.

### 5. Save the survivors — `saveCreatorsToList`

`POST /v1/creator-lists/{id}/creators` with `creator_ids` and `creator_source`
(e.g. `public`).

## Rules

- **No idempotency key exists.** `searchCreators` is billed, so a blind retry costs a
  credit. Retry only on `429` (after `Retry-After`) and `5xx`.
- Watch `X-RateLimit-Remaining`. The ceiling is 20-150 req/min depending on plan.
- `402 INSUFFICIENT_CREDITS` means top up — it is not retryable.
- `403 FORBIDDEN` on step 4 or 5 means either the plan has no workspace entitlement or
  the API key's endpoint-family scope excludes `workspace`.
- Log `meta.request_id` from every response; it is what support asks for.

## See also

- `conventions/toksta-conventions.yml` — pagination, envelope, metering
- `errors/toksta-problem-types.yml` — every error code
- `scopes/toksta-scopes.yml` — which plans reach which endpoint family
