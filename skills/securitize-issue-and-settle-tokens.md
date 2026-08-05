---
name: Fund an investment and issue digital securities on-chain
description: >-
  Take an investor from pledge through funding and subscription agreement to a recorded issuance, register their
  token wallet, and drive the prepare / sign / status blockchain transaction lifecycle.
api: openapi/securitize-domains-openapi-original.json
generated: '2026-08-05'
method: generated
operations:
  - OpportunitiesController_getOpportunities
  - RoundsController_getRounds
  - InvestmentController_createInvestment
  - InvestmentController_getInvestment
  - PledgedAmountController_updatePledgedAmount
  - PledgedAmountController_getPledgedAmount
  - FundingAddressController_getFundingAddress
  - FundedAmountController_getFundedAmount
  - SubscriptionAgreementStatusController_getSubscriptionAgreementStatus
  - SubscriptionAgreementStatusController_updateSubscriptionAgreementStatus
  - TransactionsController_createTransaction
  - TransactionsController_getTransactions
  - TokenWalletsController_addTokenWallet
  - TokenWalletsController_getTokenWallets
  - IssuancesController_createIssuance
  - IssuancesController_getIssuances
  - BlockchainTransactionsController_addBlockchainTransactionData
  - BlockchainTransactionsController_addBlockchainTransactionSign
  - BlockchainTransactionsController_updateTransactionStatus
  - BlockchainTransactionsController_getBlockchainTransactions
  - HoldersController_getHolders
---

# Fund an investment and issue digital securities on-chain

## Preconditions

The investor must already exist on the domain and be through KYC and accreditation — run
`securitize-onboard-investor.md` first. You need `domainId`, `tokenId` and the investor's `externalId`.

**This is the highest-consequence flow in the API.** It mints securities and submits blockchain transactions, and
**no operation here accepts an idempotency key.** Every write in this skill must be preceded by a read-back.

## Steps

1. **Find the opportunity and round.** `GET /v1/domains/{domainId}/opportunities/list`
   (`OpportunitiesController_getOpportunities`) and
   `GET /v1/domains/{domainId}/tokens/{tokenId}/rounds/list` (`RoundsController_getRounds`).
2. **Create the investment.** `POST /v1/domains/{domainId}/investors/{externalId}/investment`
   (`InvestmentController_createInvestment`). Read back with `InvestmentController_getInvestment` before any
   retry.
3. **Record the pledge.** `PUT .../investment/pledged-amount` (`PledgedAmountController_updatePledgedAmount`);
   confirm with `PledgedAmountController_getPledgedAmount`.
4. **Get the funding address.** `GET .../investment/funding-address`
   (`FundingAddressController_getFundingAddress`). Give this to the investor for the wire or transfer.
5. **Track funding.** `GET .../investment/funded-amount` (`FundedAmountController_getFundedAmount`). Record
   individual movements with `POST .../investment/transactions/detail`
   (`TransactionsController_createTransaction`); list with `TransactionsController_getTransactions`.
6. **Confirm the subscription agreement.** `GET .../investment/subscription-agreement-status`
   (`SubscriptionAgreementStatusController_getSubscriptionAgreementStatus`); write with the `PUT` on the same
   path (`SubscriptionAgreementStatusController_updateSubscriptionAgreementStatus`). Subscribe to the
   subscription-agreement webhook rather than polling.
7. **Register the token wallet.** `POST .../token-wallets/detail` (`TokenWalletsController_addTokenWallet`).
   Check `TokenWalletsController_getTokenWallets` first — a wallet address is the one field where a duplicate is
   both easy to create and expensive to unwind.
8. **Record the issuance.** `POST .../issuances/detail` (`IssuancesController_createIssuance`). **Read
   `IssuancesController_getIssuances` immediately before this call and immediately after.** There is no
   idempotency key; a duplicated issuance is a duplicated security.
9. **Drive the blockchain transaction.** Three ordered operations on
   `/v1/domains/{domainId}/tokens/{tokenId}/blockchain-transactions`:
   1. `POST .../prepare` (`BlockchainTransactionsController_addBlockchainTransactionData`) — build the unsigned
      transaction data.
   2. `POST .../send` (`BlockchainTransactionsController_addBlockchainTransactionSign`) — submit the signature.
   3. `PUT .../status` (`BlockchainTransactionsController_updateTransactionStatus`) — update state as the chain
      confirms.
   Track with `BlockchainTransactionsController_getBlockchainTransactions` and
   `BlockchainTransactionsController_getTransaction`.
10. **Verify the cap table.** `GET /v1/domains/{domainId}/tokens/{tokenId}/holders/list`
    (`HoldersController_getHolders`). Take a point-in-time record with `SnapshotsController_addSnapshot`.

## Corrective procedures — human approval required

These sit under `.../blockchain-transactions/procedures/` and are irreversible on-chain actions. An agent must
NOT invoke any of them without an explicit human decision recorded against the specific investor and amount:

| Operation | Effect |
|---|---|
| `ProceduresController_createClawbackTransactions` | Claw tokens back from a holder |
| `ProceduresController_createDestroyTransaction` | Burn tokens |
| `ProceduresController_createDestroyTbeTransaction` | Burn tokens held beneficially |
| `ProceduresController_createHoldTradingTransaction` | Freeze trading |
| `ProceduresController_createLostSharesTransactions` | Reissue against lost shares |
| `ProceduresController_createInternalTransferTbeTransaction` | Internal beneficial-holding transfer |
| `TransferTbeController_createTransferTbeTransaction` | Transfer beneficial holdings between investors |

See `agentic-access/securitize-agentic-access.yml` for the recommended execution contracts.

## Conventions that apply

- Errors are `{message, statusCode}`; `409 Conflict` is the signal that the record already exists — read, do not
  re-POST. See `errors/securitize-problem-types.yml`.
- Set NAV separately with `TokensController_setNav` (`POST /v1/domains/{domainId}/tokens/{tokenId}/nav`).
- Nothing in this flow is idempotent. See `conventions/securitize-conventions.yml`.
