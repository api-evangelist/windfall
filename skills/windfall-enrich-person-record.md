---
name: Enrich a person record with Windfall
description: >-
  Given a person's basic PII (name plus at least an email, full address, or
  phone+name), call the Windfall API to retrieve enriched household wealth and
  career data in real time, handling partial matches and rate limits correctly.
api: openapi/windfall-windfall-api-api-openapi.yml
operations:
- enrichRecord
---

# Enrich a person record with Windfall

Use this skill to enrich a single person record with Windfall's household (wealth)
and career data. US coverage only; data is refreshed weekly.

## Prerequisites

- An org-issued Windfall API token. Send it in the `X-WF-Auth-Token` header on
  every request. Requests must be HTTPS.
- To build and test without consuming credits, use a `sandbox_`-prefixed token
  against the sandbox base URL instead of production.

## Endpoints

- Production: `POST https://api.windfalldata.com/v1/`
- Sandbox (non-billed, deterministic personas): `POST https://api.windfalldata.com/sandbox/v1`

## Steps

1. **Assemble a matchable record.** Windfall matches when at least one combination
   resolves: an `email`; a full `address` (street number + street) **and**
   `zipcode`; or `phones` **and** `first_name` **and** `last_name`. Name alone will
   not match. Include as many identifiers as you have. Optionally set `id` to your
   own record id — it is echoed back for correlation. Add `company_name` to
   disambiguate common names for career matching.

2. **Call `enrichRecord`.** POST the JSON body to the base URL with headers
   `Content-Type: application/json` and `X-WF-Auth-Token: <token>`.

3. **Check the match flags before reading data.** The response always includes
   `household_matched` and `career_matched` (booleans). The `household` object is
   present only when `household_matched` is true; the `career` object only when
   `career_matched` is true. Never index into a nested object without checking its
   flag first.

4. **Read the enriched fields.** `household` carries up to 32 documented
   fields — `windfall_id`, `net_worth` (plus `low_confidence_net_worth` /
   `high_confidence_net_worth` bounds and `net_worth_last_calculated`),
   property flags (`multi_property_owner`, `boat_owner`, `plane_owner`), life
   events (`recent_mover`, `recently_divorced`), philanthropy
   (`philanthropic_giver`, `donor_advised_funds`), political
   (`political_donor`, `political_party`), financial signals
   (`liquidity_trigger`, `crypto_interest`) — and a `confidence` score
   (0.1–1.0). `career` carries up to 26 fields including `linkedin_url`,
   `job_title`, `job_level`, `recently_changed_jobs`, and employer
   firmographics (`company_domain`, `company_naics_code`,
   `company_size_range`). **Every field is optional**: which household fields
   return depends on the account's Windfall plan, and career fields require
   Career Intelligence (CI) on the account. Never assume a field is present.
   Full reference: `data-model/windfall-data-model.yml`.

5. **Respect the input ceiling.** Each of `emails`, `phones` and `addresses`
   accepts at most 10 entries. More than 10 in any one of them returns a `400`.
   This limit is not expressed in the OpenAPI schema, so generated clients will
   not enforce it for you.

## Error handling

- `400` — malformed body; fix the JSON.
- `401` — invalid/missing token; check the `X-WF-Auth-Token` header.
- `403` — token/environment mismatch (production token on sandbox, or vice versa).
- `429` — rate limit exceeded (max **5 requests/second**); back off and retry
  after a brief delay. For batch loops, space requests ~0.2s apart.
- `500` — unhandled server error; retry with backoff.
- `503` — downstream service unavailable; retry with backoff.

Errors use a `{ "error": "<type>", "message": "<text>" }` envelope (not RFC 9457).

**A no-match is not an error.** An unresolvable record returns `200` with both
match flags false. Only `4xx`/`5xx` are failures. Note that `403`, `500` and
`503` are documented but absent from the published OpenAPI — handle them
anyway.

## Idempotency & pagination

There is no idempotency-key header and no pagination — this is a single-record,
real-time lookup. Repeat calls re-run the lookup (sandbox responses are
deterministic per persona).
