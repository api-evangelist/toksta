---
name: Run a thought-leader discovery job
description: >-
  Start Toksta's async thought-leader search from a keyword set, poll the plain-UUID job
  endpoints, fetch one or many results, and cancel cleanly when you need to stop the spend.
api: openapi/toksta-public-api-openapi.yml
base_url: https://api.toksta.com
operations:
  - findThoughtLeaders
  - discoverySearchCreators
  - getJobStatus
  - getJobResults
  - getBulkJobResults
  - cancelJob
generated: '2026-08-13'
method: generated
source: >-
  Grounded in operationIds verified verbatim in openapi/toksta-public-api-openapi.yml;
  keyword minimum, mode pricing and reservation semantics from
  https://help.toksta.com/public-api/getting-started.
---

# Run a thought-leader discovery job

This is the one Toksta job family that uses the **generic** job endpoints, because its ids
are plain UUIDs. It is also the family where credits are *reserved* up front and settled
afterwards, so cancelling early actually saves money.

## Steps

### 1. Start the job — `findThoughtLeaders`

`POST /v1/creators/thought-leaders`

```json
{"keywords": ["B2B SaaS", "demand generation", "marketing analytics", "growth", "ABM"],
 "count": 10,
 "mode": "balanced"}
```

- **At least 5 keywords are required.**
- `mode` is `broad`, `balanced` or `strict`, costing **1, 2 or 3 credits per creator
  analyzed**.
- Credits are reserved from `max_candidates_to_analyze` × the mode rate when the job is
  accepted, and unused credits are refunded on completion.

Returns `202` with a plain-UUID `job_id`.

`discoverySearchCreators` (`POST /v1/creators/discovery-search`, 1 credit per 100
profiles) is the sibling entry point into the same job machinery when you want breadth
rather than a keyword-anchored authority search.

### 2. Poll status — `getJobStatus`

`GET /v1/jobs/{id}` with the plain UUID. The job object carries `job_type`, `status`,
`source_status`, `is_terminal`, a `progress` block (`total_items`, `completed_items`,
`approved_items`, `failed_items`), `campaign_id`, `platform`, `created_list_id`,
`result_expires_at`, `result_retention_days` and `error_message`.

Use exponential backoff — ~2s to start, capped ~30s. Stop when `is_terminal` is true.
Terminal states: `succeeded`, `failed`, `canceled`, `expired`.

### 3. Fetch results — `getJobResults` or `getBulkJobResults`

- One job: `POST /v1/jobs/results` with `{"job_id": "<uuid>", "wait_seconds": 20}`.
  `wait_seconds` (max 30) does the waiting server-side.
- Many jobs: `POST /v1/jobs/results:bulk`.

> **Path defect worth knowing:** the published OpenAPI writes the bulk route as
> `/v1/jobs/results{bulk}`, which tooling reads as an undeclared path parameter. The real
> route, per Toksta's own `GET /v1/` endpoint metadata and its docs, is
> `/v1/jobs/results:bulk` with a literal colon. Generated clients need a hand-fix.

If the job also materialized a list, `created_list_id` on the job object points at it.

### 4. Stop the spend — `cancelJob`

`POST /v1/jobs/{id}/cancel`. **A JSON body is required even when empty — send `{}`.**
Cancelling before completion releases the unspent portion of the reservation.

## Rules

- Only plain UUIDs work here. `enrichment:`, `content-fit:` and `audience-fit:` ids
  return `404` from every endpoint in this skill — use their family results endpoints.
- Do not resubmit the same start-POST unless the job failed or you deliberately want a
  second run. There is no idempotency key and a duplicate run makes a second reservation.
- Retry results calls on `5xx` and `429`; honour `Retry-After`.
- Results age out (30-day retention in the published example) and then return `404`.
- There are **no webhooks in v1**. Polling is the only completion signal.

## See also

- `conventions/toksta-conventions.yml` — the job-id format table and async rules
- `data-model/toksta-data-model.yml` — the Job entity and its relationships
- `overlays/toksta-public-api-overlay.yaml` — where the bulk-path defect is recorded
