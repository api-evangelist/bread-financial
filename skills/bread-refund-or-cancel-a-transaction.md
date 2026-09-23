---
name: bread-refund-or-cancel-a-transaction
description: Reverse a BreadPay transaction correctly — pick cancel, refund or rescind by where the transaction is in its lifecycle.
api: BreadPay Platform — Merchant Operations
generated: '2026-09-04'
method: generated
source: openapi/bread-merchant-operations-openapi.json; https://platform-docs.breadpayments.com/bread-onboarding/docs/bread-order-management
operations:
  - getTransactionByID
  - cancelTransaction
  - refundTransaction
  - rescindTransaction
---

# Reverse a BreadPay transaction

Three different operations undo a transaction and they are not interchangeable.

## Choose the operation

1. `getTransactionByID` — `GET /api/transaction/{transactionID}`. Read the current status and keep
   the `Etag`.
2. Pick by state:
   - **`cancelTransaction`** — `POST /api/transaction/{transactionID}/cancel`. Full or partial amount.
     Use before funds are released.
   - **`refundTransaction`** — `POST /api/transaction/{transactionID}/refund`. Use after settlement.
   - **`rescindTransaction`** — `POST /api/transaction/{transactionID}/rescind`. Unwinds the
     transaction itself rather than moving money back.

All three take `If-Match: <etag>` and answer **412** if the revision moved. That check is the only
replay protection on this surface.

## What is not published

Bread documents no time window for cancel, refund or rescind. There is no "refund within N days"
statement anywhere in the API reference or the merchant documentation. Treat the availability of a
reversal as something to be **read from the transaction**, never assumed from elapsed time, and
surface Bread's actual response to the user rather than a predicted one.

## Servicing-side reversals

Reversing a *transaction* is not the same as reversing a *payment* on the resulting loan. On the
Servicing side the equivalents are `refundPayment`
(`POST /api/servicing/payment-agreement/{id}/payment/refund-payment`), `cancelPayment` (pending
payments only), `cancelFuturePayment` for a scheduled payment, and the `waiveAmount` family for
fees and balances. Pick the layer that matches what actually happened.
