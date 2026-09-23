---
name: bread-authorize-and-settle-a-transaction
description: Move a BreadPay transaction from PENDING through AUTHORIZED to SETTLED, safely, using the ETag revision guard.
api: BreadPay Platform — Merchant Operations
generated: '2026-09-04'
method: generated
source: openapi/bread-merchant-operations-openapi.json; https://platform-docs.breadpayments.com/bread-onboarding/docs/bread-order-management
operations:
  - getTransactionByID
  - authorizeTransaction
  - settleTransaction
  - addFulfillmentDetail
---

# Authorize and settle a BreadPay transaction

A Bread transaction is created when the buyer completes checkout in the Bread modal and lands in
`PENDING`. It stays there until it expires unless the merchant acts. Authorizing says "the totals
match my order system"; settling releases funds and is normally sent at shipment.

**Before you start**: mint a JWT (`POST /api/auth/service/authorize`, HTTP Basic over
`apiKey:secret`) and send it as `Authorization: Bearer <jwt>`. Base URL is
`https://api.platform.breadpayments.com` (preview: `https://api-preview.platform.breadpayments.com`).

## 1. Read the transaction and capture its revision

`getTransactionByID` — `GET /api/transaction/{transactionID}`

The response carries an **`Etag`** header: the transaction revision. Keep it. This is the only replay
guard on this surface — there is no idempotency key on transaction operations.

Check the status before acting. `PENDING` is authorizable; a transaction already `AUTHORIZED` or
`SETTLED` must not be re-driven.

## 2. Authorize

`authorizeTransaction` — `POST /api/transaction/{transactionID}/authorize`

Send `If-Match: <etag from step 1>`. If the transaction changed underneath you, Bread answers **412
Precondition Failed** — re-read with `getTransactionByID` and decide again. Do not retry blindly:
without `If-Match` a repeat authorize acts a second time.

Only authorize once the order total in your OMS matches the Bread transaction amount.

## 3. Settle at shipment

`settleTransaction` — `POST /api/transaction/{transactionID}/settle`

Also `If-Match`-guarded. Settling is what releases funds to the merchant, so send it when the goods
actually ship, not at authorization.

## 4. Record fulfillment

`addFulfillmentDetail` — `POST /api/transaction/{transactionID}/fulfillment`

Carrier and tracking number, one call per shipment or an `items[]` array for a split shipment.

## Errors you must handle

| Status | Meaning | Action |
| --- | --- | --- |
| 401 | Token invalid or expired | Re-mint; tokens carry `tokenExpiresAt` |
| 403 | Token scope or merchant context wrong | Do not retry — the transaction belongs to another merchant |
| 412 | Revision mismatch | Re-read, re-decide, re-send with the fresh Etag |
| 429 | Rate limited (Checkout domain) | Blind exponential back-off — Bread publishes no Retry-After |
| 503 | Service unavailable | Back off and retry |

Errors arrive as `{reason, code, message, domain, metadata}` — not RFC 9457 problem+json.

## Undoing it

`cancelTransaction` (full or partial, pre-settlement) and `refundTransaction` (post-settlement) both
exist and are also `If-Match`-guarded. **Bread publishes no time window for either** — do not assume
one is still available at the moment you need it; read the transaction first.
