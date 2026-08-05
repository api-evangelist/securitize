---
name: Add Securitize iD sign-in and read verified investor identity
description: >-
  Run the Securitize Connect OAuth flow to let an investor sign in with Securitize iD, exchange the code for an
  access token, and read the investor information, verification details, documents, legal signers and wallets
  they consented to share.
api: 'documented endpoints only — Securitize publishes no OpenAPI for the Connect API'
generated: '2026-08-05'
method: generated
operations:
  - 'POST /v1/{clientId}/oauth2/authorize'
  - 'POST /v1/{clientId}/oauth2/refresh'
  - 'GET  /v1/{clientId}'
  - 'PATCH /v1/{clientId}'
  - 'GET  /v1/{clientId}/investor'
  - 'GET  /v1/{clientId}/investor/verification'
  - 'GET  /v1/{clientId}/investor/documents'
  - 'GET  /v1/{clientId}/investor/documents/{documentId}/view'
  - 'GET  /v1/{clientId}/investor/signers'
  - 'GET  /v1/{clientId}/investor/signers/{signerId}/documents/{documentId}/view'
  - 'GET  /v1/{clientId}/investor/domain/wallets'
  - 'POST /v1/{clientId}/investor/domain/wallets'
---

# Add Securitize iD sign-in and read verified investor identity

> **No machine-readable contract.** Unlike the Domains API, the Connect API has no OpenAPI. Its docs point at a
> Securitize iD Swagger at `https://sec-id-api.sandbox.securitize.io/swagger#/` as "the reference
> documentation" — that URL returned **404** on 2026-08-05, as did every `/swagger-json`, `/openapi.json` and
> `/api-json` variant on both hosts. Every operation below is transcribed from the prose docs at
> <https://sec-connect-api-docs.securitize.io/>. Treat request and response shapes as unverified until you have
> called them, and check for a restored Swagger before building.

## Before you start

Request from Securitize customer success:

- **`issuerId` / DomainID** — your OAuth client id, also the `{clientId}` path segment.
- **OAuth secret**.
- **Base URL** — `https://sec-id-api.securitize.io` (production) or `https://sec-id-api.sandbox.securitize.io`.

You must also give Securitize a **`redirectUrl`** to allowlist. An un-allowlisted redirect will not be honoured.
See <https://sec-connect-api-docs.securitize.io/whitelisting/whitelisting-redirected-urls>.

## Steps

1. **Send the investor to authorize.** Redirect to:

   ```
   https://id.securitize.io/#/authorize?issuerId=[CLIENT_ID]&scope=[SCOPE]&redirecturl=[REDIRECT_URL]
   ```

   `scope` is space-delimited and only three values are supported: `info details verification`. See
   `scopes/securitize-scopes.yml`.

2. **Handle the redirect.** Securitize returns to your `redirectUrl` with `code`, `country` and `authorized`:

   ```
   https://YOUR_REDIRECT?code=<uuid>&country=US&authorized=true
   ```

   **The `code` expires after 5 minutes** — exchange it server-side immediately. `authorized=true` means the
   investor had already authorized your application previously; it is absent on a first authorization, so do not
   branch on its presence alone.

3. **Exchange the code for an access token.**
   `POST https://sec-id-api.securitize.io/v1/{clientId}/oauth2/authorize`.

4. **Refresh when it expires.**
   `POST https://sec-id-api.securitize.io/v1/{clientId}/oauth2/refresh`.

5. **Read the investor.** `GET /v1/{clientId}/investor`. The response differs by investor state — the docs
   publish three worked examples: a new investor, a fully-populated individual, and an entity. Handle all three;
   do not assume the full individual shape.

6. **Read verification detail.** `GET /v1/{clientId}/investor/verification` — KYC/KYB/AML state and the
   supporting records. Requires the `verification` scope.

7. **Read documents.** `GET /v1/{clientId}/investor/documents` to list, then
   `GET /v1/{clientId}/investor/documents/{documentId}/view` to obtain a viewable/downloadable URL.

8. **Read legal signers (entities).** `GET /v1/{clientId}/investor/signers`, and their documents via
   `GET /v1/{clientId}/investor/signers/{signerId}/documents/{documentId}/view`.

9. **Wallets.** `GET /v1/{clientId}/investor/domain/wallets` to read, `POST` the same path to register one.

10. **Your own client config.** `GET /v1/{clientId}` reads your application configuration;
    `PATCH /v1/{clientId}` updates it — this is where `redirectUrls` lives.

## Rules

- **Consent is the contract.** Securitize iD returns only what the investor agreed to share for your scope. A
  missing field is a consent outcome, not an error — never re-prompt to widen scope inside a transaction.
- **No white-label KYC.** The docs are explicit: the KYC process is run and branded by Securitize. You cannot
  present it as your own flow.
- **Scope of access is also contractual**, not just technical — what your integration may see is bounded by your
  agreement with Securitize as well as by the scope string. See
  <https://sec-connect-api-docs.securitize.io/scope-of-access>.
- **PII.** Every response here is verified identity data. Do not log bodies; store only what you are permitted
  to retain.
