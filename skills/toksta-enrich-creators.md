---
name: Enrich creators and read the bill
description: >-
  Start a Toksta enrichment run for a set of creator ids, poll for results without a job
  id, and reconcile what it cost against account usage.
api: openapi/toksta-public-api-openapi.yml
base_url: https://api.toksta.com
operations:
  - startCreatorEnrichment
  - getEnrichmentResults
  - getAccountUsage
generated: '2026-08-13'
method: generated
source: >-
  Grounded in operationIds verified verbatim in openapi/toksta-public-api-openapi.yml;
  polling and job-id rules from https://help.toksta.com/public-api/getting-started.
---

# Enrich creators and read the bill

Enrichment is the one Toksta job family with **no job-status endpoint**. You track it by
re-asking the results endpoint with the same `creator_ids` you started with. Getting this
wrong — polling `GET /v1/jobs/{id}` with an `enrichment:<uuid>` id — returns `404`.

## Steps

### 1. Start the run — `startCreatorEnrichment`

`POST /v1/creators/enrich` with `{"creator_ids": ["<uuid>", ...]}`.

Enrichment is selected by **`creator_ids`**, not by `profile_url`. Get those UUIDs from
`searchCreators`, `discoverySearchCreators` or `getCreator` first.

Two possible successes:

- **`200`** with `jobs_created: 0` and `all_complete: true` — every requested creator was
  already enriched. Nothing was charged. Go straight to step 2.
- **`202`** — work was accepted. The response carries informational ids of the form
  `enrichment:<uuid>`. They are informational only; do not poll them anywhere.

Cost: **1 credit per profile**.

### 2. Fetch results — `getEnrichmentResults`

`POST /v1/enrichments/results` with the **same** `creator_ids`, plus:

- `detail_level` — e.g. `summary`
- `wait_seconds` — up to **30**, an inline server-side poll that saves you a round trip

Repeat with exponential backoff (start ~2s, cap ~30s) until every creator reports a
terminal state. Terminal states are `succeeded`, `failed`, `canceled`, `expired`.
This call costs 0 credits.

### 3. Reconcile — `getAccountUsage`

`GET /v1/account/usage` returns `period`, `total_calls`, `total_credits_charged`,
`total_overage_usd`, `plan_source`, `subscription_plan`, `subscription_status`, and an
`operations` map keyed by cost key — `profile_enrichment` is the one to read here. Each
entry gives `calls`, `credits_charged` and `overage_amount_usd`.

This is the only endpoint that tells you what a run actually cost. It is free.

## Rules

- **Never blind-retry step 1.** There is no idempotency key; a duplicate accepted run
  bills again. If step 1 times out, call step 2 first to see whether the work landed.
- Check `enrichment.is_enriched` via `getCreator` before enriching to avoid paying for
  data you already have.
- Results expire. The published job example shows `result_retention_days: 30` and a
  `result_expires_at` timestamp; past that, a fetch returns `404` and the job state is
  `expired`. Persist what you need.
- `402 INSUFFICIENT_CREDITS` on step 1 means the balance was short before any work began.
- Email finding runs inside enrichment workflows and is charged separately — the API
  pricing page says 8 credits per lookup, the marketing pricing table says 2. Verify
  against `getAccountUsage` rather than trusting either page.

## See also

- `conventions/toksta-conventions.yml` — the job-id format table
- `plans/toksta-plans-pricing.yml` — per-operation credit costs
- `data-model/toksta-data-model.yml` — the AccountUsage shape
