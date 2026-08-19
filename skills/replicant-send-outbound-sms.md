---
name: Send an outbound Replicant SMS
description: >-
  Send an SMS message to a phone number through a configured Replicant campaign. Use when an
  agent must reach a customer over messaging rather than voice — confirmations, reminders, or a
  follow-up after a call.
api: openapi/replicant-outbound-api-openapi.yml
operations:
  - sendSMS
generated: '2026-08-14'
method: generated
source: openapi/_original/replicant-outbound-api-openapi.json
---

# Send an outbound Replicant SMS

**This operation sends a real message to a real handset.** It is not reversible and Replicant
publishes no idempotency key. A retry sends a second message.

## Before you start

- **Base URL:** `https://api.replicant.ai/api/v2`
- **Auth:** `Authorization: Bearer <JWT>` (`bearerAuth`, `bearerFormat: JWT`).
- **You need a `campaignId`** — a UUID, provisioned outside this API. The campaign carries the
  channel settings and the sending number.

## Steps

1. **Use E.164 for `phone`** (`format: phone`; published example `+14083654354`).

2. **Assemble `messageData`.** Typed only as `type: object` — the free-form context the campaign
   uses to compose the message.

   > **Read this before you code the payload.** The published schema has a defect: `OutboundSMS`
   > lists `callData` in `required`, but the object defines **`messageData`** and no `callData`
   > property at all — the `required` list looks copied from `OutboundCall`. A generated client
   > will demand a field that does not exist. Send `messageData`, and confirm the exact
   > expectation with Replicant. Recorded in `overlays/replicant-outbound-api-overlay.yaml`.

3. **Call `sendSMS`.**

   ```
   POST /campaigns/{campaignId}/sms
   Authorization: Bearer <JWT>
   Content-Type: application/json

   {"phone": "+14083654354", "messageData": { ... }}
   ```

4. **Store the `messageId`.** A `201` returns `OutboundSMSResult` — `{"messageId": "<uuid>"}`.
   There is no operation to read a message back, so persist it.

5. **Do not expect a delivery receipt.** The only notification payload Replicant publishes is
   `CallStatus`, which is voice-shaped and keyed on `callId`. No SMS delivery-status schema is
   published (`asyncapi/replicant-outbound-call-status-webhooks.yml`). If you need delivery
   confirmation, agree it with Replicant during campaign setup.

## Errors

Spec declares `text/plain`; the live host returns JSON `{"error": "<message>"}`.

| Status | Meaning | What to do |
|---|---|---|
| 400 | campaignId not valid | Fix the campaign UUID — validated before auth. |
| 401 | authentication failure | Refresh the bearer token. |
| 403 | not authorized to make this request | Token not entitled to that campaign. |

See `errors/replicant-problem-types.yml`.

## Retry rule

**No idempotency key exists.** Do not auto-retry. On a timeout with no `messageId`, escalate
rather than resend — a duplicate SMS is a customer-visible failure and, in regulated messaging,
a compliance one.

## Rate limits

Undocumented; no headers returned (`rate-limits/replicant-rate-limits.yml`).
