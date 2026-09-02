# Bamboo Marketplace — Design

**Date:** 2026-09-02
**Status:** Draft for review
**License:** MIT

## 1. Problem

Bamboo Payment is a LatAm payment processor with payins, payouts, tokenization and reporting APIs. It has no marketplace features: no sub-merchants, no split payments, no per-seller balances. Every payment collected by a merchant account lands in a single pool. A marketplace that uses Bamboo must itself track how much of that pool belongs to each seller, decide when that money is available, and pay it out.

Bamboo Marketplace is a standalone, multi-tenant service that sits between a marketplace and Bamboo and provides that missing layer:

- sub-merchants with fee policies, beneficiaries and payout schedules;
- a double-entry ledger that tracks every cent from the moment Bamboo approves a payment to the moment it reaches a sub-merchant's or the marketplace's bank account;
- fund availability driven by Bamboo's own settlement data, never by assumptions;
- scheduled and on-demand payouts through Bamboo Payouts, to sub-merchants and to the marketplace itself;
- reconciliation against Bamboo with explicit, human-resolved issues;
- a REST API, signed webhooks and public documentation.

It is generic. It knows nothing about any particular marketplace's domain. The only nouns are marketplace, sub-merchant, customer, payment, refund, chargeback, payout, ledger.

The service is **not PCI compliant** and never touches card data. Card capture happens in Bamboo's hosted tokenization form; the service handles only Bamboo tokens, and even those never leave it.

## 2. How Bamboo works (confirmed with Bamboo)

- A marketplace has, **per country**, two separate Bamboo accounts: a **payins account** and a **payouts account**, each with its own credentials.
- Payments land in the payins account. Once Bamboo settles them, they become available there.
- Moving money from the payins account to the payouts account is a **manual operation performed by Bamboo's team**, on request. There is no API for it.
- All payouts, to sub-merchants and to the marketplace, are executed from the payouts account.
- Billing Movements (Bamboo's account-side ledger) can be queried for **each account**, payins and payouts, per country, using that account's own credentials. So the service sees both sides of every movement, including the manual transfer and payout fees.

- The payouts account is itself a v3 merchant account: the merchant balance endpoint and Billing Movements answer with the payouts account's private key.

### 2.1 Observed in production (Peru, 2026-09-02)

Facts taken from real responses, which override the public documentation where they differ:

- **Merchant balance** returns `{ "BalanceSummary": [ { Currency, TotalSettlement, TotalAvailable, TotalProcessing } ] }` with amounts as **decimals in major units with 4 decimal places** (e.g. `559.6822`). `TotalSettlement` is money settled but not yet available; `TotalAvailable` is payable now. Rounding noise exists (`TotalAvailable: -0.0035`).
- **Billing Movements** returns `{ "Response": { Data, Page, PageSize, Total }, "Errors" }` with amounts as **integer minor units** (`2000` = 20.00 PEN), debits negative. Field casing differs from the docs (`Transactionid`, `Movementid`, `Availabledate`, `Referenceid`, `Exchangerate`): deserialization must be case-insensitive.
- Every approved purchase produces **one `Purchase` credit row and two `TrafficFee` debit rows** (a fixed component and a percentage component), all sharing the purchase's `Transactionid` and `Created`, all with the same `Availabledate`.
- `Availabledate` was `Created + 4 calendar days` for Yape, debit and credit card alike.
- `Referenceid` is empty; the merchant's `Order` does not appear in Billing Movements. Matching is by `Transactionid` only.
- `Payment_method` and `Payment_media_brand` are sometimes swapped; neither is used for anything.
- Minimum data latency is one hour.

Everything in this design follows from those facts.

## 3. Decisions taken

| Decision | Choice | Why |
|---|---|---|
| Deployment | Standalone service, own database | Money accounting must be isolated from any product domain and reusable by many marketplaces. |
| Coverage | Full proxy: customers, tokenization, payments, refunds, webhooks, ledger, payouts | One place knows the truth about money. Marketplaces never call Bamboo directly. |
| Tenancy | Many marketplaces; each brings its own Bamboo accounts, one payins + one payouts pair per country | Mirrors Bamboo's structure. Funds of different marketplaces never mix physically. |
| Money model | Double-entry ledger, amounts in minor units, append-only | Verifiable invariants, full audit trail, refunds and chargebacks are ordinary entries. |
| Ownership model | The ledger tracks **who owns** the money sitting in Bamboo: each sub-merchant, the marketplace, or nobody yet (suspense). Revenue and expense are derived reports, not accounts. | The service is a custodian of balances. The marketplace's commission is a balance it withdraws like anyone else. |
| Fee model | Per sub-merchant: marketplace commission (percentage and/or fixed), who bears Bamboo's processing fee, whether commission is returned on refund, who bears chargeback fees | Covers every scheme a marketplace may want without special cases. |
| Availability | Only when Bamboo's Billing Movements confirms the purchase, its fee has been recorded, and `AvailableDate` has passed | Never release money on a guess. Anything stuck becomes a reconciliation issue for a human. |
| Funding payouts | Observed, not executed. The service computes how much must move to the payouts account, tells the marketplace, and records the transfer when it sees it. | Bamboo does the transfer manually. |
| Payouts | Per payee schedule plus on-demand; optional approval step; the marketplace is a payee too | Automation by default, control when wanted. |
| Countries | Configuration, not code. Initial targets: Peru, Uruguay, Mexico, Argentina | Bamboo Payouts covers all four; beneficiary field rules are country data. |
| Consumers (v1) | Server-to-server only, API keys | No UI in v1. Marketplaces build their own screens on the API. |
| Stack | .NET 8 Web API, EF Core, PostgreSQL, Hangfire | Solid transactional guarantees; the team's primary stack. |
| Docs | OpenAPI generated from code + MkDocs Material site with an embedded API reference | Public, versioned with the code, GitBook-like reading experience. |

## 4. Architecture

Single deployable: ASP.NET Core Web API hosting Hangfire for background jobs. PostgreSQL is the only datastore. Redis is not required in v1 (Hangfire uses PostgreSQL storage).

```
src/
  BambooMarketplace.Api/             controllers, auth middleware, DTOs, OpenAPI
  BambooMarketplace.Domain/          entities, ledger, fee policy, state machines, domain services
  BambooMarketplace.Infrastructure/  EF Core, repositories, Bamboo clients, crypto, Hangfire jobs
tests/
  BambooMarketplace.Domain.Tests/
  BambooMarketplace.Integration.Tests/  Testcontainers PostgreSQL
docs/                               MkDocs site
```

Dependency direction: Api → Domain ← Infrastructure. Domain has no reference to EF Core, HTTP or Bamboo. Bamboo is behind three interfaces: `IBambooPayinsClient`, `IBambooPayoutsClient` and `IBambooReportingClient` (Billing Movements, callable against either the payins or the payouts account). Every client call takes the `BambooAccount` and side whose credentials it must use.

### 4.1 Tenancy

```
Marketplace 1──* ApiKey
Marketplace 1──* BambooAccount (one per country: payins creds, payouts creds)
Marketplace 1──* MarketplaceBeneficiary (one per country)
Marketplace 1──* SubMerchant 1──1 Beneficiary
Marketplace 1──* Customer 1──* Card (token reference only)
Marketplace 1──* Payment ──* Refund, Chargeback
Marketplace 1──* Payout (payee = sub-merchant | marketplace)
Marketplace 1──* FundingTransfer
Marketplace 1──* JournalEntry 1──* Posting
Marketplace 1──* ReconciliationIssue
Marketplace 1──* OutboundEvent
```

- **Marketplace** is the tenant. Fields: name, environment (`stage` | `production`), outbound webhook URL and secret, reconciliation grace period (default 3 days), payout schedule for its own balance, status.
- **BambooAccount**: `(marketplace, country)`. Holds the payins credentials (private key, public key) and the payouts credentials (private key, digital signature), encrypted. Billing Movements is queried with each side's own credentials. Also the Bamboo merchant/account identifiers as they appear in Billing Movements, so mirrored rows can be attributed. A country without an account cannot take payments.
- **MarketplaceBeneficiary**: the marketplace's own bank account per country, so its commission can be paid out from that country's payouts account.
- **ApiKey**: many per marketplace for rotation. Stored as SHA-256 hash plus a visible prefix (`mk_live_a1b2…`). The plaintext is returned exactly once at creation. A separate admin key (`adm_…`) configured through environment is the only principal allowed on `/admin/*`.
- **SubMerchant**: name, `externalId` (marketplace's own identifier, unique per marketplace), country, currency, fee policy, payout schedule, status (`active` | `paused` | `closed`). `paused` accepts payments but generates no payouts. `closed` accepts nothing new.
- **Beneficiary**: legal name or company name, document type and number, bank account (number, type, bank code, branch, optional SWIFT), email. Required fields depend on country and are validated by a per-country rule table before Bamboo is called.
- **Customer** belongs to the marketplace, not to a sub-merchant. A customer may pay any sub-merchant of that marketplace with the same stored card. Fields: `externalId`, email, Bamboo customer id per country (Bamboo customers live inside a payins account).

Every table carries `marketplace_id`. The auth middleware resolves the marketplace from the API key and every repository query is scoped by it. There is no code path that reads another tenant's rows.

### 4.2 Secrets

Bamboo credentials are encrypted at rest with AES-256-GCM. The master key comes from the environment (`BM_MASTER_KEY`) and never from the database. Key rotation re-encrypts rows in a migration-like admin command. Logs redact anything that looks like a token, key or card number.

## 5. Ledger

Double-entry bookkeeping. The ledger scope is `(marketplace, country, currency)`: one set of books per Bamboo account pair and currency. Amounts are `long` minor units. No decimals anywhere in the money path.

### 5.1 Chart of accounts (fixed)

The ledger answers one question: of the money sitting in Bamboo, who owns what?

| Account | Kind | Meaning |
|---|---|---|
| `bamboo.payins` | asset | Money in this country's payins account |
| `bamboo.payouts` | asset | Money in this country's payouts account |
| `submerchant.{id}.pending` | liability | Owed to the sub-merchant, not yet available |
| `submerchant.{id}.available` | liability | Owed and payable |
| `submerchant.{id}.reserved` | liability | Committed to an in-flight payout |
| `marketplace.pending` | liability | Commission earned, not yet available |
| `marketplace.available` | liability | Commission payable to the marketplace |
| `marketplace.reserved` | liability | Committed to an in-flight marketplace payout |
| `suspense` | liability | Debits seen at Bamboo with no known owner yet |

There is no revenue or expense account. Commission earned, Bamboo fees paid and chargeback losses are reports computed from entry types (§5.6). Assets always equal liabilities.

### 5.2 Storage

- `journal_entries`: id, marketplace_id, country, currency, type, `source_type`, `source_id`, occurred_at, created_at, metadata (jsonb: actor, references). Unique index on `(marketplace_id, source_type, source_id)` guarantees one entry per economic fact.
- `postings`: id, entry_id, account, amount (signed; debit positive, credit negative). A database constraint and a domain check both enforce that postings of an entry sum to zero.
- `account_balances`: `(marketplace_id, country, currency, account) → balance`. Updated in the same transaction as the postings. It is a cache; the reconciliation job recomputes it from `postings` and raises an issue if they differ.

Both ledger tables are append-only. Database roles used by the application have no UPDATE or DELETE grant on them. Corrections are counter-entries.

### 5.3 Entry types and postings

Notation: Dr = debit, Cr = credit. Example: gross 10000, commission 5%, sub-merchant bears Bamboo fee.

**`payment.approved`**
Dr `bamboo.payins` 10000 · Cr `submerchant.X.pending` 9500 · Cr `marketplace.pending` 500

**`payment.bamboo_fee`** (one per `TrafficFee` row; a purchase normally has two, e.g. 24 + 326)
Cr `bamboo.payins` · Dr `submerchant.X.pending` (bearer = submerchant) **or** Dr `marketplace.pending` (bearer = marketplace). If the payment has already been released, the debit goes to the bearer's `available` instead, so a late fee row never needs special handling.

**`payment.released`** (funds available at Bamboo)
Dr `submerchant.X.pending` 9150 · Cr `submerchant.X.available` 9150
Dr `marketplace.pending` 500 · Cr `marketplace.available` 500
The marketplace's commission on a payment becomes available at the same moment as the sub-merchant's share, because it is the same money.

**`refund.completed`** (full or partial, proportional)
Cr `bamboo.payins` gross · Dr `submerchant.X.available` (falls back to `pending` if not yet released) net · Dr `marketplace.available` commission share if `refundCommission = true`, otherwise the sub-merchant bears that share too.

**`chargeback.received`**
Same shape as refund, plus the chargeback fee debited to `submerchant.X.*` or `marketplace.*` according to `chargebackFeeBearer`.

**`payout.created`** (payee P is a sub-merchant or the marketplace)
Dr `P.available` · Cr `P.reserved`

**`payout.completed`**
Dr `P.reserved` · Cr `bamboo.payouts`

**`payout.bamboo_fee`** (from the payouts account's Billing Movements, one per payout)
Cr `bamboo.payouts` · Dr `P.available` if `payoutFeeBearer = payee`, **or** Dr `marketplace.available` if the marketplace bears it. For marketplace payouts the marketplace always bears it.

**`payout.failed`** / **`payout.cancelled`**
Dr `P.reserved` · Cr `P.available`

**`funding.recorded`** (Bamboo moved money from payins to payouts; see §7.3)
Dr `bamboo.payouts` · Cr `bamboo.payins`

**`suspense.debit`** (unknown refund/chargeback/withdrawal seen in Billing Movements)
Cr `bamboo.payins` · Dr `suspense`

**`suspense.assigned`** (operator resolves an issue)
Cr `suspense` · Dr `submerchant.X.available` **or** Dr `marketplace.available`

**`manual.fee_adjustment`** (difference between an operator-entered fee and the later real fee)
Same shape as `payment.bamboo_fee`, positive or negative.

Any payee's `available` balance may go negative after refunds, chargebacks or absorbed fees. Negative balances block payouts for that payee and are carried forward until new credits cover them.

### 5.4 Fee policy

```
FeePolicy {
  commissionPercent: basis points (int)
  commissionFixed: long minor units
  bambooFeeBearer: submerchant | marketplace
  refundCommission: bool
  chargebackFeeBearer: submerchant | marketplace
  payoutFeeBearer: submerchant | marketplace
}
```

Commission = round-half-even(gross × percent) + fixed, capped at gross. Rounding happens once, in minor units. The breakdown (gross, commission, sub-merchant net) is stored on the payment and returned by the API so the marketplace never recomputes it.

### 5.5 Invariants

Checked by the reconciliation job and by tests, per `(marketplace, country, currency)`:

1. Every entry's postings sum to zero.
2. `account_balances` equals `SUM(postings)` per account.
3. `bamboo.payins` equals the payins account balance Bamboo reports (`TotalSettlement + TotalAvailable + TotalProcessing`). Additionally, the sum of all `*.pending` balances equals `TotalSettlement`, and the sum of all `*.available + *.reserved` balances plus `suspense` equals `TotalAvailable`.
4. `bamboo.payouts` equals the payouts account balance Bamboo reports.
5. `bamboo.payins + bamboo.payouts` = sum of all liability accounts.

Bamboo balances are converted from 4-decimal major units to minor units and compared with a tolerance of 1 minor unit per currency, because Bamboo's own figures carry rounding noise.

### 5.6 Derived reports

Computed from entries, never stored as balances:

- **Commission earned**: sum of `marketplace.pending` credits in `payment.approved` minus reversals in refunds/chargebacks.
- **Bamboo fees**: sum of `payment.bamboo_fee`, `payout.bamboo_fee` and `manual.fee_adjustment` amounts, split by bearer and by side (payins/payouts).
- **Chargeback losses**: sum of `chargeback.received` amounts by bearer.

Exposed on `GET /reports/…` with date filters.

## 6. Payments flow

1. Marketplace calls `POST /payments` with sub-merchant, amount, currency, customer, method and either a `card_id` or method-specific data. Country is the sub-merchant's; the payins account is the one for that country.
2. Service validates sub-merchant status, currency/country match, amount limits, and resolves `card_id` to a Bamboo commerce token by fetching the customer's payment profiles (the token is never stored locally, mirroring Bamboo's recommendation).
3. Service creates a `Payment` in `created`, then calls Bamboo Create Purchase with `Order = payment.id`, `Capture = true`, `UrlNotify` pointing back at the service.
4. On synchronous `APPROVED`, the `payment.approved` entry is posted in the same transaction that sets the payment to `approved`. On `REJECTED`, no entry. On `PENDING` (some alternative payment methods), the payment waits for the webhook.
5. Bamboo's transaction webhook arrives at `POST /webhooks/bamboo/{accountId}/transactions`. The service verifies the HMAC signature, returns 200, enqueues processing, then re-fetches the purchase by `Order` from Bamboo and applies the final status. The unique index on `(source_type, source_id)` makes the webhook and the synchronous path idempotent with respect to each other.
6. Timeouts after a request was sent: the payment stays `processing` and a job polls Get Purchase by Order every minute for 30 minutes, then hourly for 24 hours, then raises `payment_unresolved`.

Refunds: `POST /payments/{id}/refunds` calls Bamboo Refund, records the refund, posts `refund.completed` when Bamboo confirms (synchronously or by webhook). Chargebacks arrive only by webhook and post `chargeback.received`.

## 7. Reconciliation

Runs per Bamboo account, hourly by default, once against the payins side and once against the payouts side, each with its own credentials.

Four idempotent steps:

**Step 1 — Mirror.** For each side, fetch Billing Movements from `watermark − 72h` to now, paginated. Upsert each row into `bamboo_movements` keyed by `(account_id, side, MovementId)`. Raw, uninterpreted. The overlap exists because `Status` and `Withdrawal_Status` change after creation.

**Step 2 — Match.** For each new or changed movement.

Payins side:

| Type | Match by | On match | On no match |
|---|---|---|---|
| `Purchase` (Approved) | `TransactionId` → payment | store `AvailableDate`, mark `confirmed_by_bamboo` | issue `unmatched_purchase` |
| `TrafficFee` | `TransactionId` → payment | post `payment.bamboo_fee` (once per MovementId) | issue `unmatched_fee` |
| `Refund` | `TransactionId` → refund | mark confirmed | post `suspense.debit`, issue `unmatched_debit` |
| `Chargeback` | `TransactionId` → chargeback | mark confirmed | post `suspense.debit`, issue `unmatched_debit` |
| `Withdrawal` (debit) | see §7.3 | half of a funding transfer | see §7.3 |

Payouts side:

| Type | Match by | On match | On no match |
|---|---|---|---|
| incoming credit (funding) | see §7.3 | other half of a funding transfer | see §7.3 |
| payout debit | `ReferenceId`/`TransactionId` → payout | mark confirmed | post `suspense.debit` against `bamboo.payouts`, issue `unmatched_debit` |
| payout fee | → payout | post `payout.bamboo_fee` (once per MovementId) | issue `unmatched_fee` |

Bamboo's exact `Type` values on the payouts side are open question 4 in §16; the mapping is configuration, not code.

**Step 3 — Release.** A payment moves `pending → available` (entry `payment.released`) only when all three hold: its `Purchase` row is confirmed in Billing Movements, at least one `TrafficFee` row for it has been posted, and `Availabledate ≤ now`. The release amount is the payment's net of every fee posted so far; any fee row that appears later debits `available` directly (§5.3). If `Availabledate + grace period` has passed and any condition is still false, open issue `release_blocked` with the exact missing condition. Nothing is released without evidence.

**Step 4 — Check.** Query both Bamboo balances for the account and compare against the ledger: invariants 2, 3 and 4 from §5.5. Any difference opens `balance_mismatch` with the delta and both figures. Also verify that the payouts side's movements net to the same figure as the payouts balance.

### 7.1 Reconciliation issues

First-class entity: type, marketplace, account, currency, amount, references (payment, movement, payout, funding transfer), state (`open` | `resolved`), opened_at, resolved_at, resolved_by, resolution. One open issue per `(type, reference)`; repeats update `last_seen_at` instead of duplicating.

Notified to the marketplace by webhook `reconciliation.issue.opened` and listable by API. Resolution is an admin action; every resolution that moves money posts entries with `source_type = manual` and the operator identity in metadata:

| Action | Effect |
|---|---|
| `release_with_fee { amount }` | posts `payment.bamboo_fee` with the given amount and `payment.released`. When the real `TrafficFee` arrives later, posts `manual.fee_adjustment` for the difference. |
| `assign_to_submerchant { subMerchantId }` | moves a suspense debit to that sub-merchant's `available`. |
| `assign_to_marketplace` | moves a suspense debit to `marketplace.available`. |
| `confirm_funding { fundingTransferId? }` | resolves a one-sided funding movement: attaches it to the given transfer, or creates one, and posts `funding.recorded`, reversing the suspense entry. |
| `dismiss { reason }` | closes without entries. Reason is mandatory. |

### 7.2 Funding need

For each account and currency the service continuously computes:

```
funding_needed = Σ approved payouts not yet submitted
               + Σ payouts due in the next scheduling window
               − bamboo.payouts balance
```

When it is positive, the marketplace sees it on `GET /status` and receives `funding.needed` (once per change of amount, not on every run). The marketplace then asks Bamboo to move that amount.

### 7.3 Funding transfers

A `FundingTransfer` is the marketplace's record of a request to Bamboo: `POST /funding-transfers { country, currency, amount }`, state `requested`. It is bookkeeping only; the service cannot trigger the movement.

When reconciliation sees the transfer happen, it posts `funding.recorded` and marks the transfer `completed`. A transfer is evidenced by **both halves**: a withdrawal debit on the payins side and an incoming credit on the payouts side, same currency, same amount, close in time. Matching order:

1. Both halves present and equal to a `requested` transfer's amount: match it (exact match first, then the oldest requested transfer within a configurable tolerance; any difference opens `funding_amount_mismatch`).
2. Both halves present but no request: post `funding.recorded` anyway (the money moved, and both sides prove it), create the `FundingTransfer` as `completed` with `requested_by = bamboo`, and open `unrequested_funding` for information.
3. Only one half present for longer than the grace period: post `suspense.debit` (payins) or hold the credit as `suspense` (payouts) and open `unmatched_debit` / `unmatched_credit`. The operator resolves with `confirm_funding`.

The `funding.recorded` entry is only ever posted from observed evidence on both sides of the transfer.

## 8. Payouts

A payout has a **payee**: a sub-merchant or the marketplace itself. Both use the same state machine, ledger entries and API, and both are executed from the payouts account of the payee's country.

States: `pending_approval → approved → submitted → completed | failed`; `cancelled` reachable from the first two.

- **Scheduler** (hourly): for each `active` sub-merchant, and for the marketplace in each country where it has a beneficiary, whose schedule is due, with `available ≥ minimumAmount` and no payout in `approved`/`submitted`, create a payout for the full available balance. State is `approved` if `autoApprove`, else `pending_approval`.
- **Creation** (scheduler or `POST /payouts`) locks the payee row (`SELECT … FOR UPDATE`), verifies `available ≥ amount`, posts `payout.created`. Insufficient funds → 409 `INSUFFICIENT_AVAILABLE`. Missing beneficiary → 422 `BENEFICIARY_INCOMPLETE`.
- **Execution** (`approved → submitted`, every 5 minutes): check the payouts account balance at Bamboo; if insufficient, leave `approved`, open issue `payout_funds_insufficient` and let §7.2 raise the funding need. Respect the country's processing window (e.g. Uruguay business days 10:00–16:30 UTC−3) as configuration; outside it, wait for the next run. Call Create Payout with `reference = payout.id` (Bamboo's idempotency field) and `notification_Url` back to the service. Store Bamboo's `payoutId`.
- **Completion** by Bamboo's payout webhook, with a fallback poller every 15 minutes for `submitted` payouts older than one hour. `completed` posts `payout.completed`. `failed` posts `payout.failed` and stores Bamboo's error code and description.
- **Cancellation** allowed in `pending_approval` and `approved`; posts `payout.cancelled`.

## 9. API

REST over HTTPS, JSON, `snake_case` fields, amounts as integer minor units plus ISO 4217 currency. Authentication: `Authorization: Bearer <key>`. Every mutating request requires an `Idempotency-Key` header; same key and same body return the stored response, same key with a different body returns 422 `IDEMPOTENCY_KEY_REUSED`. Idempotency records expire after 24 hours.

Pagination is cursor-based (`cursor`, `limit`, `next_cursor`).

All paths below are relative to `/v1`. Inbound Bamboo webhook paths are unversioned because they are configured at Bamboo per account.

### 9.1 Admin (`adm_` key)

| Method | Path | Purpose |
|---|---|---|
| POST/GET/PATCH | `/admin/marketplaces[/{id}]` | Tenants: name, environment, webhook URL and secret, grace period, own payout schedule |
| POST/GET/PATCH/DELETE | `/admin/marketplaces/{id}/bamboo-accounts[/{country}]` | One per country: payins and payouts credentials, Bamboo identifiers |
| POST/DELETE | `/admin/marketplaces/{id}/api-keys[/{keyId}]` | Issue and revoke marketplace keys |
| GET/POST | `/admin/reconciliation/issues[/{id}/resolve]` | Cross-tenant issue list and resolution actions |

### 9.2 Marketplace (`mk_` key)

| Method | Path | Purpose |
|---|---|---|
| POST/GET/PATCH | `/submerchants[/{id}]` | Create, list, update (fee policy, schedule, status) |
| PUT | `/submerchants/{id}/beneficiary` | Bank account and document, validated per country |
| GET | `/submerchants/{id}/balance` | `{pending, available, reserved, currency}` |
| GET | `/submerchants/{id}/ledger` | Entries with postings, cursor-paginated, date filters |
| PUT/GET | `/beneficiaries/{country}` | The marketplace's own bank account per country |
| GET | `/balance` | Per country and currency: marketplace pending/available/reserved, totals owed to sub-merchants, suspense, both Bamboo asset accounts, `funding_needed` |
| GET | `/ledger` | The marketplace's own entries (its `marketplace.*` postings) |
| POST/GET | `/customers[/{id}]` | Create or fetch; returns `tokenization_session_id` for a given country |
| GET/DELETE | `/customers/{id}/cards[/{cardId}]` | List stored cards, remove one |
| GET | `/bamboo-accounts/{country}/public-key` | Bamboo public key for the hosted tokenization form |
| POST/GET | `/payments[/{id}]` | Create and read; `?reference=` lookup |
| POST | `/payments/{id}/refunds` | Full or partial refund |
| GET | `/payments/{id}/refunds`, `/payments/{id}/chargebacks` | Read |
| POST/GET | `/payouts[/{id}]` | On-demand payout; body has `payee: { type: submerchant, id } | { type: marketplace, country }` |
| POST | `/payouts/{id}/approve`, `/payouts/{id}/cancel` | Approval workflow |
| POST/GET | `/funding-transfers[/{id}]` | Record a funding request to Bamboo; read its state |
| GET | `/reports/commission`, `/reports/bamboo-fees`, `/reports/chargebacks` | Derived reports (§5.6), date-filtered |
| GET | `/reconciliation/issues` | Own issues, read-only |
| GET | `/events[/{id}]`, POST `/events/{id}/redeliver` | Outbound webhook log and manual redelivery |
| GET | `/status` | Per country: credentials present per side, last reconciliation per side, open issue counts, `funding_needed` |

### 9.3 Inbound webhooks from Bamboo

`POST /webhooks/bamboo/{accountId}/transactions`, `/chargebacks`, `/payouts`. The account id in the path selects which credentials verify the signature (payins private key for transactions and chargebacks, payouts credentials for payouts). Invalid signatures return 401 and are logged. Valid requests return 200 immediately and enqueue processing, which re-fetches the object from Bamboo by id before touching the ledger.

### 9.4 Payment request and response

```json
POST /payments
{
  "submerchant_id": "sm_…",
  "amount": 10000,
  "currency": "PEN",
  "customer_id": "cus_…",
  "method": "card",
  "card_id": "card_…",
  "reference": "order-4711",
  "description": "Order 4711",
  "client_ip": "203.0.113.7",
  "metadata": { "any": "json" }
}

201
{
  "id": "pay_…",
  "status": "approved",
  "country": "PE",
  "bamboo_transaction_id": "…",
  "amount": 10000,
  "currency": "PEN",
  "breakdown": { "gross": 10000, "commission": 500, "submerchant_net": 9500 },
  "error_code": null,
  "created_at": "…"
}
```

`method` is an open string mapped to Bamboo payment method codes per country (`card`, and Bamboo's alternative methods such as `yape` for Peru). Method-specific inputs go in `method_data`.

### 9.5 Errors

```json
{ "error": { "code": "INSUFFICIENT_AVAILABLE", "message": "…", "details": {} } }
```

Stable codes include `SUBMERCHANT_PAUSED`, `SUBMERCHANT_CLOSED`, `CURRENCY_MISMATCH`, `NO_BAMBOO_ACCOUNT_FOR_COUNTRY`, `CARD_NOT_FOUND`, `BENEFICIARY_INCOMPLETE`, `INSUFFICIENT_AVAILABLE`, `PAYOUT_NOT_CANCELLABLE`, `IDEMPOTENCY_KEY_REUSED`, `BAMBOO_UNAVAILABLE`, `BAMBOO_REJECTED` (with Bamboo's code in `details`).

## 10. Outbound webhooks

Events: `payment.approved`, `payment.rejected`, `payment.pending`, `refund.completed`, `chargeback.received`, `payout.created`, `payout.completed`, `payout.failed`, `funds.available` (payee and amount), `funding.needed`, `funding.recorded`, `reconciliation.issue.opened`.

Each event is stored in `outbound_events` with a frozen payload, then delivered by a job with headers `X-Event-Id`, `X-Event-Type`, `X-Timestamp`, `X-Signature` (HMAC SHA-256 of `timestamp.body` with the marketplace's webhook secret). Retries with exponential backoff (1m, 5m, 30m, 2h, 6h, 24h); after that the event is `dead` and can be redelivered manually. Every state is also readable by GET, so a marketplace never depends on webhooks for correctness.

## 11. Background jobs

| Job | Cadence | Purpose |
|---|---|---|
| Reconciliation | hourly per Bamboo account, both sides | §7, including funding detection and funding-need computation |
| Payout scheduler | hourly | §8, sub-merchants and marketplace |
| Payout executor | every 5 min | move `approved` → `submitted` within country windows |
| Payout status poller | every 15 min | fallback for missing payout webhooks |
| Payment resolver | every minute | fallback for payments stuck in `processing` |
| Outbound delivery | continuous | §10 |
| Webhook processor | continuous | process enqueued inbound webhooks |

Jobs are idempotent and safe to run concurrently across instances (Hangfire distributed locks per account).

## 12. Security

- No card data ever reaches the service. Tokenization uses Bamboo's hosted form; the service issues the session and public key only.
- Bamboo commerce tokens are fetched on use and never persisted. Cards are exposed to marketplaces as an opaque `card_id` derived by hashing the token.
- API keys hashed; admin key from environment; Bamboo credentials AES-GCM encrypted.
- All webhooks in and out are signed.
- Ledger tables are append-only at the database permission level.
- Structured logs with redaction; every ledger entry records its actor.

## 13. Documentation

Public docs site in `docs/`, built with MkDocs Material and published with GitHub Pages:

- Concepts: marketplaces, Bamboo accounts per country, sub-merchants, the ledger and the ownership model, availability, funding, reconciliation issues.
- Guides: onboarding a marketplace (payins and payouts credentials per country), integrating the hosted tokenization form, taking a payment, handling webhooks, paying out, requesting funding from Bamboo.
- API reference: the OpenAPI document generated from the code at build time, rendered with an embedded reference viewer, so docs cannot drift from the implementation.
- Operations: running locally, configuration, running reconciliation, resolving issues.

The API is versioned in the path (`/v1/…`). Breaking changes require a new version.

## 14. Testing

- **Domain unit tests**: property-based tests that any sequence of payment / fee / refund / chargeback / payout / funding / resolution operations leaves every entry balanced and invariants 1, 2 and 5 intact; fee rounding edge cases; state machine transitions; funding-need computation.
- **Integration tests** (Testcontainers PostgreSQL): idempotency key behaviour; two concurrent payout requests for the same payee, exactly one succeeds; reconciliation against recorded Billing Movements fixtures covering late fees, unknown purchases, console refunds, funding transfers seen on both sides, on one side only, and unrequested, status changes inside the overlap window; append-only enforcement; tenant isolation (a key never sees another marketplace's rows).
- **Bamboo clients**: contract tests against recorded responses (real production shapes from §2.1, with merchant identifiers and amounts anonymized before committing). An opt-in smoke test against Bamboo stage, excluded from CI.
- **Webhooks**: valid and invalid signatures, replay, outbound retry and dead-lettering.

## 15. Out of scope for v1

- Sub-merchant self-service portal and operations UI.
- Statement Report (CSV over SFTP/email) ingestion as an alternative reconciliation source.
- Multiple currencies within one sub-merchant.
- Split of a single payment across several sub-merchants.
- Any client-specific migration; existing Bamboo integrations move to this API as separate projects.
- Automating the funding request to Bamboo (email or ticket); v1 only computes and reports the need.

## 16. Open questions for Bamboo

1. Whether chargeback fees appear as separate Billing Movements rows and under which `Type`.
2. Exact HMAC header names and string-to-sign for the transaction, chargeback and payout webhooks in the current API version, and which key signs payout webhooks.
3. Exact `Type` values on the payouts account's Billing Movements for the incoming transfer, the payout itself and the payout fee, and whether the payout debit carries our `reference`.
4. Whether `Availabledate` can change after the movement is created, and whether T+4 (observed in Peru) differs by country.
5. Whether a refund made from the Merchant Console triggers the transaction webhook or only appears in Billing Movements.
