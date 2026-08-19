---
name: Score creator fit for a campaign
description: >-
  Create a Toksta campaign, run content-fit and audience-fit analysis against its
  targeting, and read both the scores and the post-level evidence behind them.
api: openapi/toksta-public-api-openapi.yml
base_url: https://api.toksta.com
operations:
  - createCampaign
  - getCampaign
  - listCampaigns
  - startContentMatch
  - getContentMatchResults
  - getContentMatchPosts
  - startAudienceMatch
  - getAudienceMatchResults
  - getAudienceMatchDetails
generated: '2026-08-13'
method: generated
source: >-
  Grounded in operationIds verified verbatim in openapi/toksta-public-api-openapi.yml;
  job-id and mode semantics from https://help.toksta.com/public-api/getting-started and
  https://help.toksta.com/public-api/pricing-and-plans.
---

# Score creator fit for a campaign

This is the expensive flow. Audience match runs 10-30 credits **per creator**, so shortlist
hard first (see `toksta-discover-and-shortlist-creators.md`) before you start either job.

## Steps

### 1. Establish the campaign — `createCampaign` / `listCampaigns` / `getCampaign`

`POST /v1/campaigns` with `name`, `content_keywords`, `target_audience_criteria`,
`target_platforms` and `search_country`. `GET /v1/campaigns` lists what exists;
`GET /v1/campaigns/{id}` returns one with its full targeting criteria.

Campaigns are **workspace** endpoints — SaaS entitlement only. On a dedicated API plan
these return `403`; run the analysis steps with explicit criteria instead.

### 2a. Content fit — `startContentMatch`

`POST /v1/creators/content-match`. Returns `202` with a job id of the form
`content-fit:<uuid>`. Mode controls depth and price: quick / standard / deep analyze
10 / 20 / 30 posts and cost **3 / 5 / 8 credits per creator**.

### 2b. Audience fit — `startAudienceMatch`

`POST /v1/creators/audience-match`. Returns `202` with `audience-fit:<uuid>`. Modes are
quick scan / deep dive / comprehensive at **10 / 20 / 30 credits per creator**.

### 3. Poll the right endpoint

**Do not poll `GET /v1/jobs/{id}` for either of these.** That endpoint accepts plain
UUIDs only and serves creator-discovery jobs; a `content-fit:` or `audience-fit:` id
returns `404`. Poll the family's own results endpoint:

- `getContentMatchResults` — `POST /v1/content-match/results` with `job_id` and
  `detail_level`
- `getAudienceMatchResults` — `POST /v1/audience-match/results` with `job_id` and
  `detail_level`

Both accept the **prefixed id or the bare UUID** — no need to strip the prefix. Both
accept `wait_seconds` (max 30) for an inline server-side wait. Poll while the response
reports `pending`, `queued` or `running`; stop at `succeeded`, `failed`, `canceled` or
`expired`. These reads cost 0 credits.

### 4. Read the evidence

- `getContentMatchPosts` — `POST /v1/content-match/posts` returns the post snippets the
  content score was computed from.
- `getAudienceMatchDetails` — `POST /v1/audience-match/details` returns the saved
  audience breakdown (job titles, seniority, companies of real engagers).

Always attach evidence to a recommendation. The score alone is not defensible.

## Rules

- Starter caps fit analysis at **50 content-fit and 50 audience-fit analyses per month**.
  Pro and Agency are uncapped by count but still consume credits.
- No idempotency key. A retried start-POST starts a second billed analysis. If a start
  call times out, poll the results endpoint before resubmitting.
- `cancelJob` (`POST /v1/jobs/{id}/cancel`) applies to plain-UUID discovery jobs and
  **requires a JSON body even when empty — send `{}`**.
- Back off on `429` per `Retry-After`; retry results calls on `5xx`.
- `501 NOT_IMPLEMENTED` is a declared response on every operation. Handle it: a route can
  be documented and routed before its handler is live.

## See also

- `mcp/toksta-tool-crosswalk.yml` — the same flow as MCP tools (`analyze_content_fit`,
  `analyze_audience_fit`, `get_content_fit_posts`, `get_audience_fit_details`)
- `plans/toksta-plans-pricing.yml` — mode-by-mode credit costs
- `conventions/toksta-conventions.yml` — the job-id format table
