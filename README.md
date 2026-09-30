# Global Financial Hub — Backend Architecture

This repository is the versioned canonical architecture and contract for the backend of Global Financial Hub.

It is intentionally separate from the frontend. The goal is a financially correct domain model, accounting model, persistence contract, and API contract before production backend implementation.

## Source of truth

1. `docs/DECISIONS.md`
2. `docs/DOMAIN_MODEL.md`
3. `docs/ACCOUNTING_MODEL.md`
4. `docs/DATA_MODEL.md` and `database/schema.prisma`
5. `docs/API_DESIGN.md` and `openapi/openapi.yaml`
6. examples
7. frontend implementation

The frontend is a consumer of this contract, not the backend authority.

## Core vocabulary

- **Holding:** economic thing owned or owed.
- **Position:** amount of a holding in one custody account.
- **Account:** where money/positions live: bank, broker, wallet, etc.
- **Transaction:** financial event/change.
- **Valuation:** value observation at a point in time.
- **LedgerAccount:** accounting bucket.
- **JournalEntry:** accounting event.
- **JournalLine:** debit/credit line.

A holding does not belong to exactly one account. Positions belong to accounts.

## Accounting principle

Every posted journal entry must balance: total debits = total credits.

Examples:

**Apple purchase for 1,500 USD**
- Dr Investment Asset — Apple 1,500
- Cr Cash — Broker 1,500

**Property for 200,000 EUR, financed with 50,000 cash + 150,000 mortgage**
- Dr Real Estate Asset 200,000
- Cr Cash 50,000
- Cr Mortgage Liability 150,000

**Mortgage payment of 1,000 EUR: 800 principal + 200 interest**
- Dr Mortgage Liability 800
- Dr Interest Expense 200
- Cr Cash 1,000

Transfers are not artificial sell+buy transactions.

## Financial data rules

- Money and quantities use Decimal/PostgreSQL numeric.
- Native/original currency is preserved.
- Reporting currency is derived with explicit FX.
- Missing FX is explicit unavailable/null, never silently 1.
- Acquisition cost and market valuation are different facts.
- Posted history is append-oriented; corrections use reversals/adjustments.
- API ranges use `from` inclusive and `to` exclusive.
- IDs are opaque strings/UUIDs.
- Extensible classifications are data rather than UI-only enums.

## Repository layout

```
docs/                  architecture, domain, accounting, decisions
database/schema.prisma persistence contract
openapi/openapi.yaml   API contract
examples/              canonical examples
```

## AI collaboration

Read `README.md` and `docs/DECISIONS.md` before proposing architectural changes. When uncertain, document alternatives and consequences instead of silently choosing.

Preferred order: domain -> accounting -> data -> API -> examples.

This repository is a specification/architecture layer, not yet the production backend.
