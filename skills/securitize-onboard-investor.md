---
name: Onboard an investor onto a Securitize domain
description: >-
  Add an investor to a Securitize domain, track their KYC and accreditation status, attach the documents and
  legal signers a regulated onboarding requires, and confirm they are ready to invest.
api: openapi/securitize-domains-openapi-original.json
generated: '2026-08-05'
method: generated
operations:
  - DomainsController_getDomains
  - InvestorsController_addInvestor
  - InvestorsController_addInvestors
  - InvestorsController_getInvestor
  - InvestorsController_getInvestors
  - InvestorsController_editInvestor
  - InvestorsController_sendInviteSecIdEmails
  - KycController_getKycStatus
  - KycController_updateKycStatus
  - AccreditationController_getAccreditationStatus
  - AccreditationController_updateAccreditationStatus
  - AccreditationController_sendAccreditationEmail
  - QualificationController_getQualificationStatus
  - LegalSignersController_createIndividualLegalSigner
  - LegalSignersController_createEntityLegalSigner
  - LegalSignersController_getLegalSigners
  - DocumentsController_createInvestorDocument
  - DocumentsController_getInvestorDocuments
  - LabelsController_addLabel
---

# Onboard an investor onto a Securitize domain

## Before you start

- **Base URL** — `https://public-api.securitize.io` (production) or `https://public-api.sandbox.securitize.io`
  (sandbox). Develop against sandbox: it is the only host that serves the OpenAPI.
- **Auth** — every call needs `Authorization: apiKey <keyId>:<keySecret>`. Keys are issued by Securitize customer
  success, not self-serve, and inherit one Control Panel user's permissions. There are no scopes.
- **`domainId`** — the tenant root. Read it from the Control Panel URL
  (`cp.(env).securitize.io/(Domain ID)/(Token ID)`), or call `DomainsController_getDomains` on `/v1/domains/list`.
- **`externalId`** — YOUR identifier for the investor, not one Securitize mints. Choose it, keep it stable, and
  use it in every subsequent path.

## Steps

1. **Confirm the domain.** `GET /v1/domains/list` (`DomainsController_getDomains`). Pick the `domainId` you will
   operate under. Optionally read `GeneralController_getGeneral` and `JurisdictionsController_getJurisdictions`
   first — jurisdiction rules decide whether an investor is eligible before you create them.
2. **Create the investor.** `POST /v1/domains/{domainId}/investors/detail`
   (`InvestorsController_addInvestor`). For a batch, use `POST .../investors/bulk`
   (`InvestorsController_addInvestors`) instead.
   **There is no idempotency key on this API.** If the call times out, do NOT blind-retry — read back with
   `InvestorsController_getInvestor` on `/v1/domains/{domainId}/investors/{externalId}` and only re-POST if it is
   genuinely absent.
3. **Invite them to Securitize iD.** `POST /v1/domains/{domainId}/investors/send-secid-email`
   (`InvestorsController_sendInviteSecIdEmails`). This is what starts the investor's own KYC journey.
4. **Track KYC.** Poll `GET /v1/domains/{domainId}/investors/{externalId}/kyc/status`
   (`KycController_getKycStatus`). If you run KYC yourself, write the result back with
   `KycController_updateKycStatus` (`PUT`). Prefer subscribing to the KYC/KYB webhook event over polling — see
   `asyncapi/securitize-webhooks.yml`.
5. **Track accreditation.** `GET .../accreditation/status` (`AccreditationController_getAccreditationStatus`);
   `PUT` the same path to record a determination (`AccreditationController_updateAccreditationStatus`); or send
   the investor through Securitize's own flow with
   `POST .../accreditation/send-email` (`AccreditationController_sendAccreditationEmail`).
6. **Add legal signers for entities.** `POST .../legal-signers/individual`
   (`LegalSignersController_createIndividualLegalSigner`) or `POST .../legal-signers/entity`
   (`LegalSignersController_createEntityLegalSigner`). Verify with `LegalSignersController_getLegalSigners`.
7. **Attach documents.** `POST .../documents/detail` (`DocumentsController_createInvestorDocument`); list with
   `DocumentsController_getInvestorDocuments`.
8. **Label for downstream reporting.** `POST .../labels/detail` (`LabelsController_addLabel`).
9. **Confirm readiness.** `GET .../qualification/status` (`QualificationController_getQualificationStatus`) and,
   per security, `TokenQualificationController_getTokenQualificationStatus`.

## Conventions that apply

- **Collections vs records** — collections are at `/list`, single records at `/detail` or `/{externalId}`. Do not
  assume REST-conventional collection roots.
- **Pagination** — `page` + `limit`, ordered with `orderField` + `orderDirection`, free-text via `q`. Date and
  country filters (`fromCreatedAt`, `toCreatedAt`, `countryCodes`, `investorTypes`, `labels`) are available on
  `InvestorsController_getInvestors`.
- **Errors** — the envelope is `{message, statusCode}`, not RFC 9457. Only 3 of 125 operations declare any error
  response, so treat any non-2xx as a failure you must inspect by status code. An anonymous or bad-key call
  returns a plain-text `401`, not JSON. See `errors/securitize-problem-types.yml`.
- **No rate limits are documented.** Back off on your own schedule; do not hammer the list endpoints.

## Handling personal data

Every payload in this flow is investor PII under a regulated transfer agent. Do not log request or response
bodies, do not persist identity documents outside your compliance store, and pass only the fields the operation
requires.
