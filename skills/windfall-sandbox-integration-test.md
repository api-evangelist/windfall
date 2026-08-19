---
name: Test a Windfall integration against the sandbox
description: >-
  Build and verify a Windfall API integration end to end without consuming
  credits or touching real people data — drive all four match outcomes with the
  published fictitious personas, then force 400/429/500/503 with the sandbox
  error simulator to prove your error handling works.
api: openapi/windfall-windfall-api-api-openapi.yml
operations:
- enrichRecord
---

# Test a Windfall integration against the sandbox

Windfall publishes a non-billed sandbox that mirrors production exactly — same
request body, same response shape, same matching pipeline — and differs only in
its data source. Use it as the target for every automated test. Production calls
draw down a contractual credit quota; sandbox calls do not.

## Prerequisites

- A `sandbox_`-prefixed token, issued by Windfall. Ask your account
  representative; there is no self-service provisioning.
- Base URL: `POST https://api.windfalldata.com/sandbox/v1`
- Header: `X-WF-Auth-Token: sandbox_YOUR_KEY`

Production and sandbox tokens are **not** interchangeable. A sandbox token on
production, or a production token on the sandbox, returns `403`.

## Steps

1. **Point the client at the sandbox.** Change only the base URL and the token.
   If anything else in your client has to change, your abstraction is wrong —
   the contract is identical.

2. **Cover all four match outcomes.** The sandbox holds 15 fictitious personas
   in three categories of five, spanning modest to ultra-high net worth and
   individual contributor to CEO:

   | Outcome | Personas |
   |---|---|
   | Household **and** career match | Amanda Taylor, Daniel Garcia, Michelle Robinson, James Thompson, Catherine White |
   | Household only (`career` absent) | Jane Doe, Robert Smith, Sarah Johnson, Michael Williams, Elizabeth Brown |
   | Career only (`household` absent) | David Wilson, Jennifer Davis, Thomas Anderson, Patricia Martinez, Christopher Lee |
   | No match | any record that resolves to no persona |

   Assert on the flags, not the payload: `household_matched` and
   `career_matched` are the contract. A no-match is a **`200`**, not an error —
   a test that treats `200` as "enriched" is the most common defect against this
   API.

3. **Send a genuinely matchable record.** The sandbox uses production's
   matchers, so a persona only resolves when one combination is satisfied:
   an `email`; or a full `address` (street number + street) **and** `zipcode`;
   or `phones` **and** `first_name` **and** `last_name`. Copying a single field
   out of a docs example is the usual reason a persona test stops matching.
   Extra non-matching fields are harmless.

   ```bash
   curl -X POST https://api.windfalldata.com/sandbox/v1 \
     -H "Content-Type: application/json" \
     -H "X-WF-Auth-Token: sandbox_YOUR_KEY" \
     -d '{
       "first_name": "Amanda",
       "last_name": "Taylor",
       "addresses": [{"address": "100 Demo St", "zipcode": "30301"}],
       "emails": ["amanda.taylor@sandbox.windfall.com"],
       "phones": ["5553000001"]
     }'
   ```

4. **Account for Career Intelligence.** The sandbox honors your account's
   configuration. If CI is not enabled, the "both" personas come back
   `household_matched: true` / `career_matched: false`, and the career-only
   personas come back as a full no-match. Read your account's entitlement before
   writing an assertion that expects a `career` object.

5. **Force the error paths with the simulator.** Add
   `X-Windfall-Sandbox-Error` to any sandbox request and the API returns that
   status with body `{"message": "..."}`. This is the only practical way to
   exercise `500` and `503`, which occur rarely in the sandbox naturally.

   | Header value | Status | Simulates |
   |---|---|---|
   | `400` | 400 Bad Request | invalid input |
   | `429` | 429 Too Many Requests | rate limit exceeded |
   | `500` | 500 Internal Server Error | unhandled error |
   | `503` | 503 Service Unavailable | downstream service unavailable |

   ```bash
   curl -X POST https://api.windfalldata.com/sandbox/v1 \
     -H "Content-Type: application/json" \
     -H "X-WF-Auth-Token: sandbox_YOUR_KEY" \
     -H "X-Windfall-Sandbox-Error: 429" \
     -d '{"first_name": "Jane", "last_name": "Doe"}'
   # HTTP 429  {"message": "Rate limit exceeded"}
   ```

6. **Test the naturally-occurring failures too.** The sandbox reproduces these
   under the same real conditions production does:
   `400` (malformed body, or more than 10 `emails` / `phones` / `addresses`),
   `401` (missing or invalid token), `403` (wrong token for the environment),
   `429` (over 5 req/sec). Send 11 emails to verify your client enforces the
   input ceiling before Windfall does.

7. **Verify your rate-limit discipline.** The sandbox enforces the same
   5 requests/second ceiling as production. Run your batch path against it and
   confirm you space requests (~0.2s) and retry after a `429` rather than
   hammering.

## What the sandbox does not tell you

Responses are deterministic per persona, so the sandbox cannot exercise
confidence-score variation, weekly data refresh, or the plan-dependent field set
you will actually receive. Treat every documented field as optional in
production regardless of what the sandbox returns.

## References

- Sandbox profile: `sandbox/windfall-sandbox.yml`
- Error catalog: `errors/windfall-problem-types.yml`
- Field reference: `data-model/windfall-data-model.yml`
- Windfall docs: https://api-docs.windfall.com/sandbox/
