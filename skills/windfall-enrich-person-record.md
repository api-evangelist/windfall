---
name: Enrich a person record with Windfall
description: >-
  Given a person's basic PII (name plus at least an email, full address, or
  phone+name), call the Windfall API to retrieve enriched household wealth and
  career data in real time, handling partial matches and rate limits correctly.
api: openapi/windfall-openapi-original.json
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

4. **Read the enriched fields.** `household` carries `windfall_id`, `net_worth`,
   and a `confidence` score (0.0–1.0); `career` carries `linkedin_url`,
   `job_title`, and its own `confidence`. Exact fields returned vary by account
   configuration (e.g. Career Intelligence must be enabled for career matches).

## Error handling

- `400` — malformed body; fix the JSON.
- `401` — invalid/missing token; check the `X-WF-Auth-Token` header.
- `403` — token/environment mismatch (production token on sandbox, or vice versa).
- `429` — rate limit exceeded (max **5 requests/second**); back off and retry
  after a brief delay. For batch loops, space requests ~0.2s apart.

Errors use a `{ "error": "<type>", "message": "<text>" }` envelope (not RFC 9457).

## Idempotency & pagination

There is no idempotency-key header and no pagination — this is a single-record,
real-time lookup. Repeat calls re-run the lookup (sandbox responses are
deterministic per persona).
