# Bamboo sandbox findings — 2026-09-03

Observed by calling Bamboo's staging APIs with a merchant's payins and payouts credentials for Peru. No secrets in this file. Complements the production observations in the design spec §2.1.

## Hosts

| API | Staging | Production |
|---|---|---|
| Payins (non-PCI) | `https://api.stage.bamboopayment.com/v3/api` | `https://api.bamboopayment.com/v3/api` |
| Payins (PCI, raw card) | `https://secure-api.stage.bamboopayment.com/v3/api` | `https://secure-api.bamboopayment.com/v3/api` |
| Direct tokenization (PCI) | `https://directtoken.stage.bamboopayment.com/api/Token?commerceKey=…` | `https://directtoken.bamboopayment.com/api/Token?commerceKey=…` |
| Payouts | `https://payout-api.stage.bamboopayment.com` | `https://payout-api.bamboopayment.com` |

Authentication everywhere: `Authorization: Basic <private key>` (the raw key after the word `Basic`, not base64 of `user:pass`).

## Payins

- `POST /customer` `{Email, ExternalCustomerId, FirstName, LastName}` → `{CustomerId, SessionId: "UI_…", PaymentProfiles: [], …}`. `SessionId` is what the hosted tokenization form needs. `GET /Customer/externalCustomerId/{id}` returns the same shape with a fresh `SessionId`.
- `GET /Purchase/order/{order}` returns 204 (not 404) when nothing matches.
- `POST /Purchase/{id}/Refund` with `{Amount}` for full or partial refunds; only `APPROVED` purchases are refundable.
- `GET /Merchant/balance` works with the payins key and returns `{BalanceSummary: [...]}`. In staging the figure is a placeholder (`9999999 USD`).
- `GET /Reporting/billing-movements` works with the payins key. Staging had zero rows in 90 days.
- In this sandbox merchant account, cards are disabled (`TK014 The payment method is disabled` on both the PCI purchase and direct tokenization) and Yape returns `TK007` ("payment method ID not valid for the selected country") for both `YAP` (OTP) and `YOS` (OneShot), including with the Yape test phone and OTP supplied by Bamboo. The errors are identical for every `TargetCountryISO` tried (PE, UY, MX, AR, CL, CO, BR, EC) and for every body shape, so the account simply has no payment method enabled in staging. No purchase could be created. `Purchase/Preview` fails the same way, so it cannot be used to discover enabled methods.

## Payouts

- `GET /api/Bank/country/{ISO}` (works with either key) returns `[{id, countryIsoCode, bankCode, bankName, payoutType}]`. For Peru: `payoutType 2` = bank transfer (BCP `002`, Interbank `003`, BBVA `011`, …), `payoutType 3` = wallet (Yape `901`, Plin `902`, …). Uruguay lists 15 banks.
- `POST /api/Payout/preview` returned `409 {error: {errorCode: 607, message: "The merchant account has an invalid business model"}}`.
- `POST /api/Payout` requires `DigitalSignature`. **The HMAC secret is the payouts private key itself.** String to sign is the compact JSON `{"Country":…,"Amount":…,"Currency":…,"Reference":…,"Type":…}` (PascalCase keys, in that order, `Amount` and `Type` as numbers, no spaces), HMAC SHA-256, lowercase hex. Wrong secret → bare `401`.
- Request `amount` is in **minor units** (`1000` = 10.00 PEN). `GET /api/Payout/{id}` returns `amount.value` in **major units** (`10`). The two sides of the API disagree; the ledger must convert.
- Synchronous response: `{payoutId, status, statusDescription, reference, payeeId, error}`. `payoutId` is a 64-bit integer (e.g. `354008960136820768`). Creation returns HTTP 200 for accepted payouts (`status 5 Received`) and HTTP 409 with the same body shape when declined synchronously (`status 8 Declined`, `error.errorCode`).
- Account validation is real even in staging: a 14-digit BCP account was declined with `813 Declined by validation for account`; a 20-digit CCI and a 9-digit Yape phone were accepted at creation.
- Every accepted payout then moved to `8 Declined` with `errorCode 708 Invalid business model` within seconds. The sandbox payouts account is not configured for the payouts business model. Ask Bamboo to enable it and fund the account.
- `GET /api/Payout/reference/{reference}` returns 204 when unknown.
- No balance endpoint was found on the payouts host (`/api/Balance`, `/api/Merchant/balance`, `/api/Account/balance`, `/api/Balances` → 404). `GET /v3/api/Merchant/balance` on the payins host with the payouts key works in production but returned `400 TR996 internal error` in staging.
- Status codes: 1 PAID (final), 2 PENDING, 3 PROCESSING, 4 REJECTED (final), 5 RECEIVED, 6 VALIDATED, 7 HELD (manual compliance review), 8 DECLINED (final).
- Payout webhook payload mirrors `GET /api/Payout/{id}` plus `payoutType`. Signature: HMAC SHA-256 over `country + amount + currency + reference + payoutType + payoutId` with the private key.

## 2026-09-04 — a second staging payins account, with methods enabled

Bamboo supplied a different staging merchant key with Yape and cards enabled, plus Yape test data (phone `969929157`, OTP `557454`).

- `POST /Purchase` with `PaymentMethod: "YAP"` and `MetaDataIn {phoneNumber, otp}` → `200 {TransactionId, Result: "COMPLETED", Status: "APPROVED", AuthorizationCode, Amount, Currency, Url: ".../v3/api/transaction/{id}", PaymentMethod: {Brand: "Yape", Type: "BankTransfer"}}`. Yape OTP is synchronous.
- The PCI raw-card purchase on `secure-api.stage` also approved (VISA test card), after one transient `504 TR999 An error occurred while processing the workflow`. Retry on 5xx is mandatory.
- `POST /Purchase/{id}/Refund {Amount: 500}` → `200 {TransactionId (new), Result: "COMPLETED", Status: "PENDING", AuthorizationCode: "DeferredRefundPending", Amount: -500}`. Thirty seconds later `GET /transaction/{refundId}` showed `Type: "REFUND", Status: "APPROVED"`. Refunds are asynchronous with a separate transaction id.
- `GET /transaction/{id}` is a generic read for any transaction type and adds a `Type` field (`PURCHASE`, `REFUND`); `GET /Purchase/{id}` and `GET /Purchase/order/{order}` return the purchase shape without `Type`.
- **This account settles in USD** although it charges in PEN. Its Billing Movements rows are all `Currency: "USD"` with `Exchangerate ≈ 3.35`; the merchant balance is in USD. A third movement type appears: **`EfExSpread`** (FX spread, Debit). Per purchase: one `Purchase` credit, one `TrafficFee` debit (3.0%, no fixed component), one `EfExSpread` debit (~10%). `Availabledate = Created + 20 days` (19 across a DST-free boundary once), not the 4 days of the production Peru account.
- The same key authenticates on the payouts host and creates payouts (`status 5 Received`), but they are declined seconds later with `708 Invalid business model`, exactly like the dedicated payouts key. Payouts need an account with the payouts business model regardless of which key is used.
- **Do not generalize from this.** Bamboo confirmed that in production the two accounts are strict: the payins account only takes payments and the payouts account only sends payouts. A staging key accepting calls on both hosts is a sandbox convenience. The design keeps two credential sets per country and never uses one for the other's operations.

### Today's transactions in Billing Movements (about 1.5 h later)

- **Query window quirk.** With `To` = current UTC time the API returned zero rows for transactions created almost two hours earlier; with `To` = tomorrow it returned all 15. The `To` bound is evaluated against something ahead of UTC (or exclusive of the current day). Always query with `To` at least one day in the future.
- **Latency.** The merchant balance reflected the purchases within seconds; Billing Movements showed them between one and two hours later.
- **Refund rows.** The refund appears as `Type: "Refund"`, Debit, with the **refund's own `Transactionid`** (the one returned by `POST /Purchase/{id}/Refund`), not the purchase's. `Exchangerate` is `null` on the refund row. `Availabledate` equals the **original purchase's** `Availabledate`.
- **Refunds carry fees.** The refund `Transactionid` also has its own `TrafficFee` (−4 USD cents on a 149-cent refund, 3%) and `EfExSpread` (−14, ~10%) rows. Refunding is not free; the ledger must post fee entries for refunds too.
- **Card vs Yape**: identical fee structure on this account (3% `TrafficFee`, ~10% `EfExSpread`, `Availabledate = Created + 20 days`).
- **Rounding.** Sum of today's movements: +2300 USD cents net. Balance moved from 222.2258 to 245.1855 = +22.9597. Difference 0.0403 USD across five transactions: Bamboo keeps sub-cent FX fractions in the balance while movements are whole cents. Reconciliation tolerance must scale with the number of movements, not be a flat one cent.

### Payout `reference` is not enforced as unique by the API

`POST /api/Payout` accepted the same `reference` three times (twice in one account, once in another), returning a new `payoutId` each time. `GET /api/Payout/reference/{ref}` then returned only the first one. The declined state of the earlier payout may be what allowed the repeat (uniqueness might apply only to live payouts), but the service cannot rely on that. Consequences: our own idempotency is the only real guard; after a timeout on create, look up by reference before retrying, and never treat "same reference" as "same payout". Declined payouts are returned by both `GET by id` and `GET by reference`.

The Merchant Console's bulk upload, by contrast, rejects rows whose `Reference` already exists in that console's account with the message "Referencia repetida", for every row of the file.

## Payins webhooks (from docs, not yet observed)

- Transaction webhook: signed with HMAC SHA-256 over `PurchaseId + Amount + Currency + utcNow`, where `utcNow` comes from a `dateSent` header; the signature travels in a header whose name the public docs do not state. Retries: 15m, 30m, 1h, 3h, 6h.
- Chargeback webhook: `{chargebackId, transactionId, order, amount, currency, status: PENDING|APPROVED|REJECTED, created, reasonCode, description}`. No fee field.

## Blocked until Bamboo acts

1. Enable a payment method (cards or Yape) on the staging payins account, with test data for Yape if that is the method.
2. Enable the payouts business model on the staging payouts account and fund it.
3. Confirm the header names for the payins webhook signature and `dateSent`.
4. Confirm how to read the payouts account balance (the `/v3/api/Merchant/balance` route with the payouts key, as in production, or something else).

Once 1 and 2 are done, the plan is: create purchases and payouts, wait for Billing Movements (one-hour latency) on both accounts, and record the `Type` values for the incoming funding, the payout debit and the payout fee.
