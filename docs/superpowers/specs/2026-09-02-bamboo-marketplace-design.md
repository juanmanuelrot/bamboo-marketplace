# Bamboo Marketplace — Design

**Date:** 2026-09-02
**Status:** Draft for review
**License:** MIT

## 1. Problem

Bamboo Payment is a LatAm payment processor with payins, payouts, tokenization and reporting APIs. It has no marketplace features: no sub-merchants, no split payments, no per-seller balances. Every payment collected by a merchant account lands in a single pool. A marketplace that uses Bamboo must itself track how much of that pool belongs to each seller, decide when that money is available, and pay it out.

Bamboo Marketplace is a standalone, multi-tenant service that sits between a marketplace and Bamboo and provides that missing layer:

- sub-merchants with fee policies, beneficiaries and payout schedules;
- a double-entry ledger that tracks every cent from the moment Bamboo approves a payment to the moment it reaches a sub-merchant's bank account;
- fund availability driven by Bamboo's own settlement data, never by assumptions;
- scheduled and on-demand payouts through Bamboo Payouts;
- reconciliation against Bamboo with explicit, human-resolved issues;
- a REST API, signed webhooks and public documentation.

It is generic. It knows nothing about any particular marketplace's domain. The only nouns are marketplace, sub-merchant, customer, payment, refund, chargeback, payout, ledger.

The service is **not PCI compliant** and never touches card data. Card capture happens in Bamboo's hosted tokenization form; the service handles only Bamboo tokens, and even those never leave it.

## 2. Decisions taken

| Decision | Choice | Why |
|---|---|---|
| Deployment | Standalone service, own database | Money accounting must be isolated from any product domain and reusable by many marketplaces. |
| Coverage | Full proxy: customers, tokenization, payments, refunds, webhooks, ledger, payouts | One place knows the truth about money. Marketplaces never call Bamboo directly. |
| Tenancy | Many marketplaces; each brings its own Bamboo merchant account credentials | Funds of different marketplaces never mix physically. Each marketplace is Bamboo's customer; the service operates on its behalf. |
| Money model | Double-entry ledger, amounts in minor units, append-only | Verifiable invariants, full audit trail, refunds and chargebacks are ordinary entries. |
| Fee model | Per sub-merchant: marketplace commission (percentage and/or fixed), who bears Bamboo's processing fee, whether commission is returned on refund, who bears chargeback fees | Covers every scheme a marketplace may want without special cases. |
| Availability | Only when Bamboo's Billing Movements confirms the purchase, its fee has been recorded, and `AvailableDate` has passed | Never release money on a guess. Anything stuck becomes a reconciliation issue for a human. |
| Payouts | Per sub-merchant schedule plus on-demand; optional approval step | Automation by default, control when wanted. |
| Countries | Configuration, not code. Initial targets: Peru, Uruguay, Mexico, Argentina | Bamboo Payouts covers all four; beneficiary field rules are country data. |
| Consumers (v1) | Server-to-server only, API keys | No UI in v1. Marketplaces build their own screens on the API. |
| Stack | .NET 8 Web API, EF Core, PostgreSQL, Hangfire | Solid transactional guarantees; the team's primary stack. |
| Docs | OpenAPI generated from code + MkDocs Material site with an embedded API reference | Public, versioned with the code, GitBook-like reading experience. |

## 3. Architecture

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

Dependency direction: Api → Domain ← Infrastructure. Domain has no reference to EF Core, HTTP or Bamboo. Bamboo is behind two interfaces: `IBambooPayinsClient` and `IBambooPayoutsClient`, plus `IBambooReportingClient` for Billing Movements.

### 3.1 Tenancy

```
Marketplace 1──* ApiKey
Marketplace 1──1 BambooCredentials (payins, payouts, reporting)
Marketplace 1──* SubMerchant 1──1 Beneficiary
Marketplace 1──* Customer 1──* Card (token reference only)
Marketplace 1──* Payment ──* Refund, Chargeback
SubMerchant  1──* Payout
Marketplace 1──* JournalEntry 1──* Posting
Marketplace 1──* ReconciliationIssue
Marketplace 1──* OutboundEvent
```

- **Marketplace** is the tenant. Fields: name, environment (`stage` | `production`), Bamboo credentials (encrypted), outbound webhook URL and secret, reconciliation grace period (default 3 days), status.
- **ApiKey**: many per marketplace for rotation. Stored as SHA-256 hash plus a visible prefix (`mk_live_a1b2…`). The plaintext is returned exactly once at creation. A separate admin key (`adm_…`) configured through environment is the only principal allowed on `/admin/*`.
- **SubMerchant**: name, `externalId` (marketplace's own identifier, unique per marketplace), country, currency, fee policy, payout schedule, status (`active` | `paused` | `closed`). `paused` accepts payments but generates no payouts. `closed` accepts nothing new.
- **Beneficiary**: legal name or company name, document type and number, bank account (number, type, bank code, branch, optional SWIFT), email. Required fields depend on country and are validated by a per-country rule table before Bamboo is called.
- **Customer** belongs to the marketplace, not to a sub-merchant. A customer may pay any sub-merchant of that marketplace with the same stored card. Fields: `externalId`, email, Bamboo customer id.

Every table carries `marketplace_id`. The auth middleware resolves the marketplace from the API key and every repository query is scoped by it. There is no code path that reads another tenant's rows.

### 3.2 Secrets

Bamboo credentials are encrypted at rest with AES-256-GCM. The master key comes from the environment (`BM_MASTER_KEY`) and never from the database. Key rotation re-encrypts rows in a migration-like admin command. Logs redact anything that looks like a token, key or card number.

## 4. Ledger

Double-entry bookkeeping per marketplace and currency. Amounts are `long` minor units. No decimals anywhere in the money path.

### 4.1 Chart of accounts (fixed)

| Account | Kind | Meaning |
|---|---|---|
| `bamboo.clearing` | asset | Money Bamboo holds for this marketplace from payins |
| `bamboo.payout_funds` | asset | Money in the marketplace's Bamboo Payouts balance |
| `submerchant.{id}.pending` | liability | Owed to the sub-merchant, not yet available |
| `submerchant.{id}.available` | liability | Owed and payable |
| `submerchant.{id}.reserved` | liability | Committed to an in-flight payout |
| `marketplace.revenue` | income | Commission earned |
| `marketplace.bamboo_fees` | expense | Bamboo fees the marketplace chose to absorb |
| `marketplace.suspense` | liability | Debits seen at Bamboo with no known owner yet |

### 4.2 Storage

- `journal_entries`: id, marketplace_id, currency, type, `source_type`, `source_id`, occurred_at, created_at, metadata (jsonb: actor, references). Unique index on `(marketplace_id, source_type, source_id)` guarantees one entry per economic fact.
- `postings`: id, entry_id, account, amount (signed; debit positive, credit negative). A database constraint and a domain check both enforce that postings of an entry sum to zero.
- `account_balances`: `(marketplace_id, currency, account) → balance`. Updated in the same transaction as the postings. It is a cache; the reconciliation job recomputes it from `postings` and raises an issue if they differ.

Both ledger tables are append-only. Database roles used by the application have no UPDATE or DELETE grant on them. Corrections are counter-entries.

### 4.3 Entry types and postings

Notation: Dr = debit, Cr = credit. Example: gross 10000, commission 5%, sub-merchant bears Bamboo fee.

**`payment.approved`**
Dr `bamboo.clearing` 10000 · Cr `submerchant.X.pending` 9500 · Cr `marketplace.revenue` 500

**`payment.bamboo_fee`** (from Billing Movements `TrafficFee`, e.g. 350)
Cr `bamboo.clearing` 350 · Dr `submerchant.X.pending` 350 (bearer = submerchant) **or** Dr `marketplace.bamboo_fees` 350 (bearer = marketplace)

**`payment.released`** (net now available)
Dr `submerchant.X.pending` 9150 · Cr `submerchant.X.available` 9150

**`refund.completed`** (full or partial, proportional)
Cr `bamboo.clearing` gross · Dr `submerchant.X.available` (falls back to `pending` if not yet released) net · Dr `marketplace.revenue` commission share if `refundCommission = true`, otherwise the sub-merchant bears that share too.

**`chargeback.received`**
Same shape as refund, plus the chargeback fee debited to `submerchant.X.*` or `marketplace.bamboo_fees` according to `chargebackFeeBearer`.

**`payout.created`**
Dr `submerchant.X.available` · Cr `submerchant.X.reserved`

**`payout.completed`**
Dr `submerchant.X.reserved` · Cr `bamboo.payout_funds`

**`payout.failed`** / **`payout.cancelled`**
Dr `submerchant.X.reserved` · Cr `submerchant.X.available`

**`payout_funding.recorded`** (admin action: money moved from payins balance to payouts balance at Bamboo)
Dr `bamboo.payout_funds` · Cr `bamboo.clearing`

**`suspense.debit`** (unknown refund/chargeback/withdrawal seen in Billing Movements)
Cr `bamboo.clearing` · Dr `marketplace.suspense`

**`suspense.assigned`** (operator resolves an issue)
Cr `marketplace.suspense` · Dr `submerchant.X.available` **or** Dr `marketplace.bamboo_fees`

**`manual.fee_adjustment`** (difference between an operator-entered fee and the later real fee)
Same shape as `payment.bamboo_fee`, positive or negative.

A sub-merchant's `available` balance may go negative after refunds or chargebacks. Negative balances block payouts and are carried forward until new payments cover them.

### 4.4 Fee policy

```
FeePolicy {
  commissionPercent: decimal (basis points stored as int)
  commissionFixed: long minor units
  bambooFeeBearer: submerchant | marketplace
  refundCommission: bool
  chargebackFeeBearer: submerchant | marketplace
}
```

Commission = round-half-even(gross × percent) + fixed, capped at gross. Rounding happens once, in minor units. The breakdown (gross, commission, sub-merchant net) is stored on the payment and returned by the API so the marketplace never recomputes it.

### 4.5 Invariants

Checked by the reconciliation job and by tests:

1. Every entry's postings sum to zero.
2. `account_balances` equals `SUM(postings)` per account.
3. `bamboo.clearing` equals Bamboo's merchant balance (settlement + available + processing) per currency.
4. `bamboo.payout_funds` equals Bamboo's payouts balance.
5. Sum of all `submerchant.*` balances + `marketplace.*` balances = `bamboo.clearing` + `bamboo.payout_funds`.

## 5. Payments flow

1. Marketplace calls `POST /payments` with sub-merchant, amount, currency, country, customer, method and either a `cardId` or method-specific data.
2. Service validates sub-merchant status, currency/country match, amount limits, and resolves `cardId` to a Bamboo commerce token by fetching the customer's payment profiles (the token is never stored locally, mirroring Bamboo's recommendation).
3. Service creates a `Payment` in `created`, then calls Bamboo Create Purchase with `Order = payment.id`, `Capture = true`, `UrlNotify` pointing back at the service.
4. On synchronous `APPROVED`, the `payment.approved` entry is posted in the same transaction that sets the payment to `approved`. On `REJECTED`, no entry. On `PENDING` (some alternative payment methods), the payment waits for the webhook.
5. Bamboo's transaction webhook arrives at `POST /webhooks/bamboo/{marketplaceId}/transactions`. The service verifies the HMAC signature, returns 200, enqueues processing, then re-fetches the purchase by `Order` from Bamboo and applies the final status. The unique index on `(source_type, source_id)` makes the webhook and the synchronous path idempotent with respect to each other.
6. Timeouts after a request was sent: the payment stays `processing` and a job polls Get Purchase by Order every minute for 30 minutes, then hourly for 24 hours, then raises `payment_unresolved`.

Refunds: `POST /payments/{id}/refunds` calls Bamboo Refund, records the refund, posts `refund.completed` when Bamboo confirms (synchronously or by webhook). Chargebacks arrive only by webhook and post `chargeback.received`.

## 6. Reconciliation

Runs per marketplace, hourly by default. Requires the marketplace's Billing Movements credentials (issued by Bamboo's Technical Account Manager). Without them, nothing is ever released automatically and the marketplace sees `RECONCILIATION_CREDENTIALS_MISSING` on its status endpoint.

Four idempotent steps:

**Step 1 — Mirror.** Fetch Billing Movements from `watermark − 72h` to now, paginated. Upsert each row into `bamboo_movements` keyed by `(marketplace_id, MovementId)`. Raw, uninterpreted. The overlap exists because `Status` and `Withdrawal_Status` change after creation.

**Step 2 — Match.** For each new or changed movement:

| Type | Match by | On match | On no match |
|---|---|---|---|
| `Purchase` (Approved) | `TransactionId` → payment | store `AvailableDate`, mark `confirmed_by_bamboo` | issue `unmatched_purchase` |
| `TrafficFee` | `TransactionId` → payment | post `payment.bamboo_fee` (once per MovementId) | issue `unmatched_fee` |
| `Refund` | `TransactionId` → refund | mark confirmed | post `suspense.debit`, issue `unmatched_debit` |
| `Chargeback` | `TransactionId` → chargeback | mark confirmed | post `suspense.debit`, issue `unmatched_debit` |
| `Withdrawal` | `Withdrawal_Id` → recorded funding | mark confirmed | post `suspense.debit`, issue `unmatched_debit` |

**Step 3 — Release.** A payment moves `pending → available` (entry `payment.released`) only when all three hold: confirmed by Bamboo in Billing Movements, its fee entry exists, and `AvailableDate ≤ now`. If `AvailableDate + grace period` has passed and any condition is still false, open issue `release_blocked` with the exact missing condition. Nothing is released without evidence.

**Step 4 — Check.** Compare ledger against Bamboo: invariants 3, 4 and 2 from §4.5. Any difference opens `balance_mismatch` with the delta and both figures.

### 6.1 Reconciliation issues

First-class entity: type, marketplace, currency, amount, references (payment, movement, payout), state (`open` | `resolved`), opened_at, resolved_at, resolved_by, resolution. One open issue per `(type, reference)`; repeats update `last_seen_at` instead of duplicating.

Notified to the marketplace by webhook `reconciliation.issue.opened` and listable by API. Resolution is an admin action; every resolution that moves money posts entries with `source_type = manual` and the operator identity in metadata:

| Action | Effect |
|---|---|
| `release_with_fee { amount }` | posts `payment.bamboo_fee` with the given amount and `payment.released`. When the real `TrafficFee` arrives later, posts `manual.fee_adjustment` for the difference. |
| `assign_to_submerchant { subMerchantId }` | moves a suspense debit to that sub-merchant's `available`. |
| `absorb_by_marketplace` | moves a suspense debit to `marketplace.bamboo_fees`. |
| `dismiss { reason }` | closes without entries. Reason is mandatory. |

## 7. Payouts

States: `pending_approval → approved → submitted → completed | failed`; `cancelled` reachable from the first two.

- **Scheduler** (hourly): for each `active` sub-merchant whose schedule is due, with `available ≥ minimumAmount` and no payout in `approved`/`submitted`, create a payout for the full available balance. State is `approved` if `autoApprove`, else `pending_approval`.
- **Creation** (scheduler or `POST /payouts`) locks the sub-merchant row (`SELECT … FOR UPDATE`), verifies `available ≥ amount` and no negative carry, posts `payout.created`. Insufficient funds → 409 `INSUFFICIENT_AVAILABLE`. Missing beneficiary → 422 `BENEFICIARY_INCOMPLETE`.
- **Execution** (`approved → submitted`): check Bamboo Payouts balance; if insufficient, leave `approved` and open issue `payout_funds_insufficient`. Respect the country's processing window (e.g. Uruguay business days 10:00–16:30 UTC−3) as configuration; outside it, wait for the next run. Call Create Payout with `reference = payout.id` (Bamboo's idempotency field) and `notification_Url` back to the service. Store Bamboo's `payoutId`.
- **Completion** by Bamboo's payout webhook, with a fallback poller every 15 minutes for `submitted` payouts older than one hour. `completed` posts `payout.completed`. `failed` posts `payout.failed` and stores Bamboo's error code and description.
- **Cancellation** allowed in `pending_approval` and `approved`; posts `payout.cancelled`.

Funding the payouts balance from the payins balance is an operational step at Bamboo whose mechanics are not documented publicly. The service records it (`POST /admin/marketplaces/{id}/payout-funding`) and verifies it against Bamboo's balances; it does not perform it. **Open question for Bamboo:** whether the Withdrawal endpoint or a manual process moves settled payins funds into the payouts balance.

## 8. API

REST over HTTPS, JSON, `snake_case` fields, amounts as integer minor units plus ISO 4217 currency. Authentication: `Authorization: Bearer <key>`. Every mutating request requires an `Idempotency-Key` header; same key and same body return the stored response, same key with a different body returns 422 `IDEMPOTENCY_KEY_REUSED`. Idempotency records expire after 24 hours.

Pagination is cursor-based (`cursor`, `limit`, `next_cursor`).

All paths below are relative to `/v1`. Inbound Bamboo webhook paths are unversioned because they are configured at Bamboo per marketplace.

### 8.1 Admin (`adm_` key)

| Method | Path | Purpose |
|---|---|---|
| POST/GET/PATCH | `/admin/marketplaces[/{id}]` | Tenants, Bamboo credentials, webhook URL and secret, grace period, environment |
| POST/DELETE | `/admin/marketplaces/{id}/api-keys[/{keyId}]` | Issue and revoke marketplace keys |
| POST | `/admin/marketplaces/{id}/payout-funding` | Record payins → payouts funding |
| GET/POST | `/admin/reconciliation/issues[/{id}/resolve]` | Cross-tenant issue list and resolution actions |

### 8.2 Marketplace (`mk_` key)

| Method | Path | Purpose |
|---|---|---|
| POST/GET/PATCH | `/submerchants[/{id}]` | Create, list, update (fee policy, schedule, status) |
| PUT | `/submerchants/{id}/beneficiary` | Bank account and document, validated per country |
| GET | `/submerchants/{id}/balance` | `{pending, available, reserved, currency}` |
| GET | `/submerchants/{id}/ledger` | Entries with postings, cursor-paginated, date filters |
| POST/GET | `/customers[/{id}]` | Create or fetch; returns `tokenization_session_id` |
| GET/DELETE | `/customers/{id}/cards[/{cardId}]` | List stored cards, remove one |
| GET | `/public-key` | Bamboo public key for the hosted tokenization form |
| POST/GET | `/payments[/{id}]` | Create and read; `?reference=` lookup |
| POST | `/payments/{id}/refunds` | Full or partial refund |
| GET | `/payments/{id}/refunds`, `/payments/{id}/chargebacks` | Read |
| POST/GET | `/payouts[/{id}]` | On-demand payout, list with filters |
| POST | `/payouts/{id}/approve`, `/payouts/{id}/cancel` | Approval workflow |
| GET | `/balance` | Marketplace aggregates per currency |
| GET | `/reconciliation/issues` | Own issues, read-only |
| GET | `/events[/{id}]`, POST `/events/{id}/redeliver` | Outbound webhook log and manual redelivery |
| GET | `/status` | Health of this tenant: credentials present, last reconciliation, open issue counts |

### 8.3 Inbound webhooks from Bamboo

`POST /webhooks/bamboo/{marketplaceId}/transactions`, `/chargebacks`, `/payouts`. Signature verified with HMAC SHA-256 and the marketplace's private key as Bamboo documents; invalid signatures return 401 and are logged. Valid requests return 200 immediately and enqueue processing, which re-fetches the object from Bamboo by id before touching the ledger.

### 8.4 Payment request and response

```json
POST /payments
{
  "submerchant_id": "sm_…",
  "amount": 10000,
  "currency": "PEN",
  "country": "PE",
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
  "bamboo_transaction_id": "…",
  "amount": 10000,
  "currency": "PEN",
  "breakdown": { "gross": 10000, "commission": 500, "submerchant_net": 9500 },
  "error_code": null,
  "created_at": "…"
}
```

`method` is an open string mapped to Bamboo payment method codes per country (`card`, and Bamboo's alternative methods such as `yape` for Peru). Method-specific inputs go in `method_data`.

### 8.5 Errors

```json
{ "error": { "code": "INSUFFICIENT_AVAILABLE", "message": "…", "details": {} } }
```

Stable codes include `SUBMERCHANT_PAUSED`, `SUBMERCHANT_CLOSED`, `CURRENCY_MISMATCH`, `CARD_NOT_FOUND`, `BENEFICIARY_INCOMPLETE`, `INSUFFICIENT_AVAILABLE`, `PAYOUT_NOT_CANCELLABLE`, `IDEMPOTENCY_KEY_REUSED`, `RECONCILIATION_CREDENTIALS_MISSING`, `BAMBOO_UNAVAILABLE`, `BAMBOO_REJECTED` (with Bamboo's code in `details`).

## 9. Outbound webhooks

Events: `payment.approved`, `payment.rejected`, `payment.pending`, `refund.completed`, `chargeback.received`, `payout.created`, `payout.completed`, `payout.failed`, `submerchant.funds_available`, `reconciliation.issue.opened`.

Each event is stored in `outbound_events` with a frozen payload, then delivered by a job with headers `X-Event-Id`, `X-Event-Type`, `X-Timestamp`, `X-Signature` (HMAC SHA-256 of `timestamp.body` with the marketplace's webhook secret). Retries with exponential backoff (1m, 5m, 30m, 2h, 6h, 24h); after that the event is `dead` and can be redelivered manually. Every state is also readable by GET, so a marketplace never depends on webhooks for correctness.

## 10. Background jobs

| Job | Cadence | Purpose |
|---|---|---|
| Reconciliation | hourly per marketplace | §6 |
| Payout scheduler | hourly | §7 |
| Payout executor | every 5 min | move `approved` → `submitted` within country windows |
| Payout status poller | every 15 min | fallback for missing payout webhooks |
| Payment resolver | every minute | fallback for payments stuck in `processing` |
| Outbound delivery | continuous | §9 |
| Webhook processor | continuous | process enqueued inbound webhooks |

Jobs are idempotent and safe to run concurrently across instances (Hangfire distributed locks per marketplace).

## 11. Security

- No card data ever reaches the service. Tokenization uses Bamboo's hosted form; the service issues the session and public key only.
- Bamboo commerce tokens are fetched on use and never persisted. Cards are exposed to marketplaces as an opaque `card_id` derived by hashing the token.
- API keys hashed; admin key from environment; Bamboo credentials AES-GCM encrypted.
- All webhooks in and out are signed.
- Ledger tables are append-only at the database permission level.
- Structured logs with redaction; every ledger entry records its actor.

## 12. Documentation

Public docs site in `docs/`, built with MkDocs Material and published with GitHub Pages:

- Concepts: marketplaces, sub-merchants, the ledger, availability, reconciliation issues.
- Guides: onboarding a marketplace (including which Bamboo credentials to request), integrating the hosted tokenization form, taking a payment, handling webhooks, paying out.
- API reference: the OpenAPI document generated from the code at build time, rendered with an embedded reference viewer, so docs cannot drift from the implementation.
- Operations: running locally, configuration, running reconciliation, resolving issues.

The API is versioned in the path (`/v1/…`). Breaking changes require a new version.

## 13. Testing

- **Domain unit tests**: property-based tests that any sequence of payment / fee / refund / chargeback / payout / resolution operations leaves every entry balanced and invariants 1, 2 and 5 intact; fee rounding edge cases; state machine transitions.
- **Integration tests** (Testcontainers PostgreSQL): idempotency key behaviour; two concurrent payout requests for the same sub-merchant, exactly one succeeds; reconciliation against recorded Billing Movements fixtures covering late fees, unknown purchases, console refunds, status changes inside the overlap window; append-only enforcement.
- **Bamboo clients**: contract tests against recorded stage responses. An opt-in smoke test against Bamboo stage, excluded from CI.
- **Webhooks**: valid and invalid signatures, replay, outbound retry and dead-lettering.

## 14. Out of scope for v1

- Sub-merchant self-service portal and operations UI.
- Statement Report (CSV over SFTP/email) ingestion as an alternative reconciliation source.
- Multiple currencies within one sub-merchant.
- Split of a single payment across several sub-merchants.
- Any client-specific migration; existing Bamboo integrations move to this API as separate projects.

## 15. Open questions for Bamboo

1. How settled payins funds are moved into the Payouts balance (Withdrawal endpoint vs. manual operation), and the timing.
2. Whether `TrafficFee` movements always carry the purchase's `TransactionId`.
3. Whether chargeback fees appear as separate Billing Movements rows and under which `Type`.
4. Exact HMAC header names and string-to-sign for the transaction, chargeback and payout webhooks in the current API version.
