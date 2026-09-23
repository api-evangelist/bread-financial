---
name: bread-subscribe-to-webhooks
description: Create, verify and operate a BreadPay webhook subscription, including JWS signature verification and failed-event replay.
api: BreadPay Platform — webhook
generated: '2026-09-04'
method: generated
source: openapi/bread-webhook-openapi.json; https://platform-docs.breadpayments.com/bread-developers/docs/webhook-concepts
operations:
  - createSubscription
  - getAllSubscriptions
  - getClientJWKs
  - createSubscriptionTest
  - getEventBySubscriptionAndStatus
  - replayEventBySubscriptionNameAndEventID
  - deleteSubscription
---

# Subscribe to BreadPay webhooks

Bread delivers events as **CloudEvents 1.0.2 over the HTTP protocol binding**, signed with a detached
JWS. Subscriptions are fully self-service through the API.

## 1. Create the subscription

`createSubscription` — `POST /api/webhook/subscription`

Body needs `name`, `subscriptionURL`, `eventSubscriptions[]` and `auth`. The `auth` object is a union:
`{"authType":"NONE"}`, `{"authType":"BASIC", username, password}`, or a static bearer token. Bread
sends that credential to *your* endpoint so you can authenticate the delivery.

`eventSubscriptions[]` takes exact event names or a wildcard over a namespace — the published example
is `breadpayments.webhooks.my.super-rev1`.

> **Gap to plan around**: Bread's API reference lists webhooks per domain under a "Webhooks" section,
> but those sections render empty and no event-type enumeration appears in any of the nine published
> OpenAPI definitions. You will need the event names from your Bread integration manager. Do not
> guess them.

## 2. Verify every delivery

Fetch the signing keys once and cache them: `getClientJWKs` — `GET /api/webhook/client/jwks`.

Each POST to your endpoint carries:

- `X-Jws-Signature` — detached JWS, `typ: JOSE`, with a `kid` matching one of those JWKs and a
  `crit: [Timestamp]` header
- `ce-id`, `ce-source`, `ce-type`, `ce-specversion: 1.0`, `ce-time`
- `timestamp` — time of *sending*, not of the original event
- `User-Agent: BreadPay-Webhooks/2.0`

Route on `ce-source` **plus** `ce-type`; `ce-type` alone is not unique across domains. Dedupe on
`ce-id`, which is a UUID per triggering event — deliveries can repeat.

Every payload carries an `Identity` object with at least `tenantID`; use it to attribute the event to
the right merchant.

## 3. Prove it end to end

`createSubscriptionTest` — `POST /api/webhook/subscription/{name}/integration-test-event`

Fires a synthetic `integration_test_event` at your endpoint. This is the safe way to exercise the
whole path without creating real financing activity.

## 4. Operate it

- `getEventBySubscriptionAndStatus` — `GET /api/webhook/subscription/{name}/event`, filterable by
  `SUCCESS | FAILURE | PENDING`. Poll for `FAILURE` after an outage.
- `replayEventBySubscriptionNameAndEventID` —
  `POST /api/webhook/subscription/{name}/event/{id}/dispatch`. Redelivers one event. Your handler
  must be idempotent on `ce-id`, because replay is exactly the case where you will see a duplicate.
- `deleteSubscription` — `DELETE /api/webhook/subscription/{name}` reverses step 1.
