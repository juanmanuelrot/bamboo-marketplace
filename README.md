# Bamboo Marketplace

A multi-tenant marketplace layer on top of [Bamboo Payment](https://docs.bamboopayment.com).

Bamboo has no sub-merchants, no split payments and no per-seller balances: everything a merchant collects lands in one pool. This service gives any marketplace the missing pieces:

- sub-merchants with fee policies, beneficiaries and payout schedules;
- a double-entry ledger tracking every cent from Bamboo approval to a sub-merchant's bank account;
- fund availability driven by Bamboo's own settlement data, never by assumptions;
- scheduled and on-demand payouts through Bamboo Payouts;
- reconciliation against Bamboo with explicit, human-resolved issues;
- a REST API, signed webhooks and public documentation.

It is generic: the only concepts are marketplace, sub-merchant, customer, payment, refund, chargeback, payout and ledger. It is **not PCI compliant** and never touches card data; cards are captured by Bamboo's hosted tokenization form.

## Status

Design phase. The current design is in [`docs/superpowers/specs/2026-09-02-bamboo-marketplace-design.md`](docs/superpowers/specs/2026-09-02-bamboo-marketplace-design.md).

## Stack

.NET 8, EF Core, PostgreSQL, Hangfire. Documentation with MkDocs Material and an OpenAPI reference generated from the code.

## License

Open source. License to be chosen before the first release.
