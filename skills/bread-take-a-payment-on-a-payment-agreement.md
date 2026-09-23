---
name: bread-take-a-payment-on-a-payment-agreement
description: Take a payment against a BreadPay payment agreement using the one operation on the platform that supports an idempotency key.
api: BreadPay Platform — Servicing
generated: '2026-09-04'
method: generated
source: openapi/bread-servicing-openapi.json
operations:
  - listBuyerPaymentAgreements
  - getPaymentAgreement
  - calculatePayments
  - makePayment
  - refundPayment
---

# Take a payment on a BreadPay payment agreement

Servicing is the post-purchase side: the loan created at checkout, its balance, and the money moving
against it.

## 1. Find the agreement

`listBuyerPaymentAgreements` — `GET /api/servicing/buyer/{buyerID}/payment-agreement`, then
`getPaymentAgreement` — `GET /api/servicing/payment-agreement/{id}` for the detail. List endpoints
across the platform page with `limit` / `offset` / `sort` and filter with `<field>.<operator>` query
parameters (`status.eq`, `createdAt.gte`, …).

## 2. Work out the amount

`calculatePayments` — `GET /api/servicing/calculated-payments` before committing to a figure, rather
than computing schedules yourself.

## 3. Take the payment — with the idempotency key

`makePayment` — `POST /api/servicing/payment-agreement/{id}/payment`

**Send `xIdempotencyKey`.** It is optional in the contract and it is the *only* idempotency key on the
entire BreadPay Platform — one operation out of roughly ninety mutating ones. A retry without it is a
second payment. Bread does not publish the key's retention period, so treat the safe-retry window as
short and unknown.

Watch `PaymentStatus` on the way back: `SCHEDULED`, `PENDING`, `RECEIVED`, `FAILED`, `CANCELLED`,
`CHARGED_BACK`, `REFUNDED`, `RETURNED`, `REJECTED`. Only `RECEIVED` is money in; `PENDING` is not
settled and ACH results can arrive days later as `RETURNED`.

## 4. Reversing it

- `cancelPayment` — `DELETE /api/servicing/payment-agreement/{id}/payment/{paymentID}/cancel-payment`,
  **pending payments only**.
- `refundPayment` — `POST /api/servicing/payment-agreement/{id}/payment/refund-payment`, creates an
  offsetting refund.
- `cancelFuturePayment` for something scheduled but not yet taken.

No time window is published for any of these. Read the payment's current status before you attempt a
reversal rather than inferring from how long ago it was taken.

## Handling the money

Errors come back as `{reason, code, message, domain, metadata}`. Servicing reason codes are
`Servicing_PaymentAgreement_BadRequest`, `Servicing_PaymentAgreement_InvalidIdentity` and
`Servicing_PaymentAgreement_ServerError` — a 500-class `ServerError` on a payment call is ambiguous
about whether the payment landed. Re-read the agreement's payments before retrying, and always retry
with the same `xIdempotencyKey`.
