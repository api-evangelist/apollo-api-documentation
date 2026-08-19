---
name: apollo-enroll-contacts-in-a-sequence
description: Find or create an Apollo sequence, pick the sending mailbox, enroll contacts, then watch activity and stop the ones that reply.
api: apollo-api-documentation
base_url: https://api.apollo.io/api/v1
operations:
  - search-for-sequences
  - create-sequence
  - get-a-list-of-email-accounts
  - add-contacts-to-sequence
  - get-contact-sequence-activity
  - update-contact-status-sequence
  - approve-sequence
  - abort-sequence
generated: '2026-08-14'
method: generated
source: openapi/_original/apollo-api-documentation-apollo-rest-api-openapi.json
---

# Enroll contacts in a sequence

Outreach is the one place where an agent can do real, irreversible damage on a customer's behalf —
enrolling the wrong contacts sends real email from a real mailbox. Treat every step here as
approval-required.

## Steps

1. **Find the sequence.** `POST /emailer_campaigns/search` (`search-for-sequences`).
   To build one instead, `POST /sequences` (`create-sequence`). Create it **inactive** so a human can
   review the steps before anything sends, then activate deliberately with
   `POST /emailer_campaigns/{sequence_id}/approve` (`approve-sequence`).

2. **Pick the sending mailbox — do this before enrolling.**
   `GET /email_accounts` (`get-a-list-of-email-accounts`) returns the connected mailboxes.
   Enrollment fails if you have no connected email account, and the mailbox you pick is the identity
   the prospect sees.

3. **Enroll contacts.** `POST /emailer_campaigns/{sequence_id}/add_contact_ids`
   (`add-contacts-to-sequence`) with `contact_ids[]`, the chosen `send_email_from_email_account_id`,
   and `emailer_campaign_id`. Contacts must already exist — enrich and save them first
   (see `apollo-enrich-and-save-a-prospect`).

4. **Watch the response.** `POST /emailer_campaigns/activity_feed`
   (`get-contact-sequence-activity`) returns per-contact sequence activity.
   `GET /emailer_messages/{id}/activities` (`get_emailstats`) returns per-message stats.

5. **Stop outreach when someone replies.**
   `POST /emailer_campaigns/remove_or_stop_contact_ids` (`update-contact-status-sequence`).
   To halt the whole sequence, `POST /emailer_campaigns/{sequence_id}/abort` (`abort-sequence`);
   to retire it, `POST /emailer_campaigns/{sequence_id}/archive` (`archive-sequence`).

## Rules

- Sequence operations cost 0 credits. The credit risk in this flow is upstream, in enrichment.
- Rate limits are the plan defaults — 200/minute, 400/hour, 2,000/day on Basic and Professional;
  600/hour and 6,000/day on Organization. Free is 50/200/600.
- There is no idempotency key. Re-POSTing `add_contact_ids` after a timeout can double-enroll;
  read the activity feed to confirm state before retrying.
- Apollo MCP exposes the same flow as tools but rate-limits it separately and returns no
  `x-rate-limit-*` headers.
