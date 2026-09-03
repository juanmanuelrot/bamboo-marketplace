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
- In this sandbox merchant account, cards are disabled (`TK014 The payment method is disabled` on both the PCI purchase and direct tokenization) and Yape returns `TK007 The payment method does not match the expected` for both `YAP` (OTP) and `YOS` (OneShot). No purchase could be created. Ask Bamboo to enable at least one method in staging.

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

## Payins webhooks (from docs, not yet observed)

- Transaction webhook: signed with HMAC SHA-256 over `PurchaseId + Amount + Currency + utcNow`, where `utcNow` comes from a `dateSent` header; the signature travels in a header whose name the public docs do not state. Retries: 15m, 30m, 1h, 3h, 6h.
- Chargeback webhook: `{chargebackId, transactionId, order, amount, currency, status: PENDING|APPROVED|REJECTED, created, reasonCode, description}`. No fee field.

## Blocked until Bamboo acts

1. Enable a payment method (cards or Yape) on the staging payins account, with test data for Yape if that is the method.
2. Enable the payouts business model on the staging payouts account and fund it.
3. Confirm the header names for the payins webhook signature and `dateSent`.
4. Confirm how to read the payouts account balance (the `/v3/api/Merchant/balance` route with the payouts key, as in production, or something else).

Once 1 and 2 are done, the plan is: create purchases and payouts, wait for Billing Movements (one-hour latency) on both accounts, and record the `Type` values for the incoming funding, the payout debit and the payout fee.
