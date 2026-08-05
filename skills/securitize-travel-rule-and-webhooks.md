---
name: Register Travel Rule counterparties and subscribe to investor events
description: >-
  Use the Securitize Travel Rule partner API to register individual and entity investors and issue blockchain
  ids, then wire real-time investor events by creating webhook subscriptions with a delivery signature.
api: openapi/securitize-domains-openapi-original.json
generated: '2026-08-05'
method: generated
operations:
  - InvestorsController_createTrIndividualInvestor
  - InvestorsController_getTrIndividualInvestor
  - InvestorsController_createTrEntityInvestor
  - InvestorsController_getTrEntityInvestor
  - InvestorsController_getTrInvestor
  - InvestorsController_createInvestorBlockchainId
  - InvestorsController_getInvestorBlockchainId
  - EventsController_getEvents
  - SubscriptionsController_createSubscription
  - SubscriptionsController_getSubscriptions
  - SubscriptionsController_getSubscription
  - SubscriptionsController_updateSubscription
  - SubscriptionsController_deleteSubscription
  - SettingsController_createSignature
---

# Register Travel Rule counterparties and subscribe to investor events

Two surfaces that partners — rather than domain operators — most often need. Both are in the Domains API and
both use the same `Authorization: apiKey <keyId>:<keySecret>` header.

## Part 1 — Travel Rule registration

The `/v1/tr/` namespace implements FATF Travel Rule counterparty registration. It is keyed by `securitizeId`,
not by the operator's `externalId` — a different identity space from the rest of the API.
Docs: <https://domain-api-docs.securitize.io/guide/tr-api-for-partners>

1. **Register an individual.** `POST /v1/tr/domains/{domainId}/investors/individual`
   (`InvestorsController_createTrIndividualInvestor`).
2. **Register an entity.** `POST /v1/tr/domains/{domainId}/investors/entity`
   (`InvestorsController_createTrEntityInvestor`).
3. **Issue a blockchain id.** `POST /v1/tr/domains/{domainId}/investors/blockchain-id`
   (`InvestorsController_createInvestorBlockchainId`).
4. **Read back.** `GET /v1/tr/domains/{domainId}/investors/{securitizeId}`
   (`InvestorsController_getTrInvestor`), or the type-specific
   `InvestorsController_getTrIndividualInvestor` / `InvestorsController_getTrEntityInvestor` /
   `InvestorsController_getInvestorBlockchainId`.

**These three POSTs are the only operations in the whole API that declare error responses**, and they are the
place the no-idempotency rule bites hardest:

- `400` → `{message, statusCode}` (`TrInvestorErrorResponseDto`). Validation failure. Fix the payload.
  The blockchain-id variant returns `BlockchainIdErrorResponseDto`, which adds `name` and a `data` member
  echoing the partial result — inspect `data` before retrying.
- `409` → the investor or blockchain id **already exists**. This is what a blind retry produces. Do not treat
  409 as failure: issue the matching `GET` and use the existing record.

## Part 2 — Webhook subscriptions

1. **Discover the event catalog.** `GET /v1/webhooks/events` (`EventsController_getEvents`) returns
   `WebhookEventDto[]` — each an `eventType` string plus the `properties` names its payload carries.
   **This is auth-gated; there is no public event list.** Securitize's marketing page names three categories —
   KYC/KYB updates, accreditation updates, subscription agreement updates — but the actual `eventType` values
   only come from this call. Never hardcode an event name you have not read from this endpoint.
2. **Create the signing secret first.** `POST /v1/webhooks/settings/signature`
   (`SettingsController_createSignature`). Do this before subscribing so your first delivery is verifiable.
   Securitize does not publish the signature algorithm or header name — get them from customer success.
3. **Subscribe.** `POST /v1/webhooks/subscriptions` (`SubscriptionsController_createSubscription`) with
   `{domainId, eventType, payloadUrl, isActive}`. `isActive` defaults to `true`.
4. **Manage.** `SubscriptionsController_getSubscriptions` to list;
   `SubscriptionsController_getSubscription`, `SubscriptionsController_updateSubscription` (PATCH) and
   `SubscriptionsController_deleteSubscription` on `/v1/webhooks/subscriptions/{subscriptionId}`.

The subscription record carries a `nonce` (a number) alongside `id`, `domainId`, `eventType`, `payloadUrl` and
`isActive` — use it for replay protection.

## Why this matters

Polling `KycController_getKycStatus` and
`SubscriptionAgreementStatusController_getSubscriptionAgreementStatus` across a book of investors is the
default failure mode on this API, and there are no documented rate limits to tell you when you have gone too
far. Subscribe instead. See `asyncapi/securitize-webhooks.yml`.
