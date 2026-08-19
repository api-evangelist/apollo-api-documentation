---
name: apollo-check-budget-before-a-bulk-job
description: Read Apollo's two independent budgets — rate limits and credits — before running a bulk enrichment or search job, so the job does not stall on a 429 or silently burn a month of credits.
api: apollo-api-documentation
base_url: https://api.apollo.io/api/v1
operations:
  - get-current-user-profile
  - post_apiusage
  - view-credit-usage-stats
  - bulk-people-enrichment
  - bulk-organization-enrichment
generated: '2026-08-14'
method: generated
source: openapi/_original/apollo-api-documentation-apollo-rest-api-openapi.json, https://docs.apollo.io/reference/rate-limits, https://docs.apollo.io/docs/api-pricing
---

# Check budget before a bulk job

Apollo meters two things independently and an agent must respect both. A call can be well inside its
rate limit and still be a commercial mistake because it spends credits.

## Steps

1. **Confirm which user you are acting as.** `GET /users/api_profile`
   (`get-current-user-profile`). Costs 0 credits. With an API key you act as the workspace's
   longest-standing active admin, and every record you create is owned by that user. The returned `id`
   is the value that will appear in `object_owner_id`.

2. **Read the live rate limits for this team.** `POST /usage_stats/api_usage_stats`
   (`post_apiusage`). This is the source of truth — the published table covers current plans only, and
   legacy workspaces differ. The response reports usage and limits per API route.

3. **Read the credit balance.** `POST /usage_stats/credit_usage_stats`
   (`view-credit-usage-stats`).

4. **Cost the job before running it.**
   - Record management (create/update/list/search-your-own-records): **0 credits**.
   - `people-enrichment` / `bulk-people-enrichment`: **1–9 credits per person** — 1 for
     demographics/email, +8 if a mobile phone is returned. Bulk takes up to 10 people per request.
   - Waterfall email typically 1–4 credits and can exceed 20; waterfall phone typically 8–25 and can
     exceed 45, depending on the vendors configured. Some vendors charge even on a miss.
   - `organization-enrichment` / `bulk-organization-enrichment` / `get_organizations{id}` /
     `get_people{id}`: **1 credit each**.
   - `organization-search`: **1 credit per page** (≤100 results). `news_articles_search`: 1 credit per
     page (≤25). `organization-jobs-postings`: 1 credit per page (≤10,000).
   - Paging multiplies the charge. Test with one small request, verify the spend at step 3, then scale.

5. **Run the job and read the headers on every response.**
   `x-rate-limit-minute` / `-hourly` / `-24-hour` are your ceilings;
   `x-minute-requests-left` / `x-hourly-requests-left` / `x-24-hour-requests-left` are the remaining
   budget. An empty `x-rate-limit-*` means that window is not limited for this endpoint.

6. **On `429`,** read `retry-after` — it is the exact number of seconds until the exhausted window
   resets. Do not guess a backoff; Apollo publishes the number.

## Gotchas

- Limits are per **team** and per **endpoint**, and windows are fixed but not clock-aligned: a window
  opens on your first request. Child workspaces share the parent's limits.
- Rate-limit headers appear only after authentication and authorization succeed. A `401` or `403`
  carries none.
- `sync-report` (Query Analytics Report) is capped at **5 requests per hour** on every plan.
- Apollo MCP uses separate limits and returns none of these headers.
