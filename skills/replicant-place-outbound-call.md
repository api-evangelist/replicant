---
name: Place an outbound Replicant call
description: >-
  Trigger a Replicant AI voice agent to call a phone number using a configured campaign, then
  correlate the outcome from the call-status notification. Use when an agent must initiate a real
  outbound phone conversation (reminder, collection, confirmation, roadside dispatch) through
  Replicant.
api: openapi/replicant-outbound-api-openapi.yml
operations:
  - placeCall
generated: '2026-08-14'
method: generated
source: openapi/_original/replicant-outbound-api-openapi.json
---

# Place an outbound Replicant call

**This operation dials a real human being.** It is not reversible, not idempotent, and Replicant
publishes no idempotency key. Treat every call to it as a physical-consequence action: confirm
the phone number and the campaign before you send, and never retry blindly on a timeout.

## Before you start

- **Base URL:** `https://api.replicant.ai/api/v2`
- **Auth:** `Authorization: Bearer <JWT>` — the spec's only security scheme (`bearerAuth`,
  `bearerFormat: JWT`). Credentials are issued by Replicant to enterprise customers; there is no
  self-serve signup. See `authentication/replicant-authentication.yml`.
- **You need a `campaignId`.** Campaigns are provisioned outside this API — there is no list,
  read, or create campaign operation on the public surface. Get the UUID from your Replicant
  implementation contact or the dashboard.

## Steps

1. **Validate the number before you send.** `phone` is `format: phone` and every published
   example is E.164 (`+14083654354`). Send E.164.

2. **Assemble `callData`.** It is typed only as `type: object` — free-form, and it is the
   context handed to the AI agent for the conversation (account number, appointment time, order
   id, whatever the campaign script expects). The campaign definition, not the spec, determines
   which keys are required. Confirm the shape with the campaign owner.

3. **Call `placeCall`.**

   ```
   POST /campaigns/{campaignId}/calls
   Authorization: Bearer <JWT>
   Content-Type: application/json

   {"phone": "+14083654354", "callData": { ... }}
   ```

   Both `phone` and `callData` are required.

4. **Store the `callId`.** A `201` returns `OutboundCallResult` — `{"callId": "<uuid>"}`. That id
   is the only handle you get; there is no operation to read a call back. Persist it before you
   do anything else.

5. **Wait for the call-status notification.** The API is fire-and-callback: the spec's own
   description is "placing outbound calls through Replicant, and being notified of call status",
   and the `CallStatus` payload carries `status`, `callCompleted`, `callFailed`, `callStartedAt`,
   `callDuration`, `callSuccessAt` and free-form `callResults`. Match on `callId`. Treat
   `callCompleted` / `callFailed` as the terminal signals; `status` is a free-form string with no
   published vocabulary. See `asyncapi/replicant-outbound-call-status-webhooks.yml` — note the
   subscription mechanism is configured during campaign setup, not through this API.

## Errors

The spec declares `text/plain` bodies, but the production host returns JSON —
`{"error": "<message>"}`. Parse JSON, and fall back to text.

| Status | Meaning | What to do |
|---|---|---|
| 400 | `Invalid campaign UUID` / campaignId not valid | Fix the campaign UUID. This is checked **before** auth, so you can hit it with no token. |
| 401 | authentication failure | Refresh the bearer token. |
| 403 | not authorized to make this request | The token is valid but not entitled to that campaign. |

There is no declared `404`, no declared `429`, and no declared `5xx`. See
`errors/replicant-problem-types.yml`.

## Retry rule

**Do not auto-retry a non-2xx or a timeout.** No idempotency key exists
(`conventions/replicant-conventions.yml`), so a retry places a second call to the same person. On
a timeout with no `callId`, escalate to a human rather than resending.

## Rate limits

None are published — no `RateLimit-*` headers, no `Retry-After`, no documented ceiling
(`rate-limits/replicant-rate-limits.yml`). The only published capacity boundary is a plan limit:
**10 concurrent calls on Quick Start**, unlimited on Professional and Enterprise. Pace yourself
against your plan's concurrency, not against a header.
