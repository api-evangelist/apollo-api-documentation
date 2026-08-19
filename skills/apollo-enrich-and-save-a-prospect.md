---
name: apollo-enrich-and-save-a-prospect
description: Find a person in Apollo's database, enrich them for verified contact details, and convert them into a saved Contact so the workspace never pays credits for that record again.
api: apollo-api-documentation
base_url: https://api.apollo.io/api/v1
operations:
  - people-api-search
  - people-enrichment
  - create-a-contact
  - search-for-contacts
  - view-credit-usage-stats
generated: '2026-08-14'
method: generated
source: openapi/_original/apollo-api-documentation-apollo-rest-api-openapi.json, https://docs.apollo.io/docs/convert-enriched-people-to-contacts
---

# Enrich and save a prospect

Apollo charges credits for enrichment but not for saved contacts. The whole point of this flow is to
pay once: enrich, then persist the result as a Contact so the next lookup is free.

## Before you start

- Authenticate with `x-api-key: <key>` (Apollo users) or an OAuth bearer token (partners).
  Sending an Apollo API key as a Bearer token returns `401 Invalid API key`.
- Free accounts registered with a personal email address cannot call search or enrichment at all.
  Register with a work email.

## Steps

1. **Check for an existing contact first.** `POST /contacts/search` (`search-for-contacts`).
   This costs 0 credits. If the person is already saved, stop — you have the record.

2. **Search the database.** `POST /mixed_people/api_search` (`people-api-search`) with the filters you
   have: title, seniority, department, organization domain, location, employee count.
   Costs 0 credits. Results carry profile and availability data, **not** email addresses or phone
   numbers — that is what step 3 is for.
   - Pass `page` and `per_page` to walk results. There is no cursor.
   - Industry filters expect Apollo industry tag IDs, not free text. Free text returns `422`.

3. **Enrich the match.** `POST /people/match` (`people-enrichment`) with the strongest identifier you
   have — `email`, `linkedin_url`, or `first_name` + `last_name` + `domain`.
   - Costs 1 credit for demographics/email, plus 8 more if a mobile phone is returned.
   - Set `reveal_phone_number=true` only when you actually need the phone; it is the expensive half.
   - For waterfall enrichment set `run_waterfall_email=true` and/or `run_waterfall_phone=true` **and**
     supply `webhook_url`. Apollo answers immediately with a status and a request id, then POSTs the
     final result to your webhook. Your receiver must be HTTPS and must tolerate duplicate deliveries —
     Apollo retries. If you miss the callback, `GET /webhook_result/{request_id}` (`poll-webhook-result`).

4. **Save the person as a Contact.** `POST /contacts` (`create-a-contact`) with the enriched fields.
   Costs 0 credits. From here the record is permanently accessible to the workspace and re-reading it
   never spends credits again.
   - Set `owner_id` explicitly if you do not want ownership to default to the acting user. With an API
     key the acting user is the workspace's longest-standing active admin, not whoever made the key.

5. **Confirm the spend.** `POST /usage_stats/credit_usage_stats` (`view-credit-usage-stats`).

## Rate limits and errors

- Enrichment on paid plans: 1,000 requests/minute per team per endpoint, no hourly or daily cap.
  On Free: 50/minute (20 for bulk), 200/hour, 600/day.
- On `429`, read the `retry-after` header for the seconds until the exhausted window resets. Budget
  from `x-minute-requests-left` / `x-hourly-requests-left` / `x-24-hour-requests-left`.
- `403` means the endpoint is outside your plan or key scope — not a malformed request.
- Apollo does **not** support an idempotency key. Do not blind-retry `POST /contacts`; re-run the
  step 1 search instead.
