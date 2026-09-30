# API Design

Base path: `/api/v1`

## Conventions
- Opaque string/UUID IDs.
- ISO-8601 dates.
- `from` inclusive, `to` exclusive.
- Money: `{"amount":"1500.00","currency":"USD"}`.
- Quantities are decimal strings.
- Consistent errors: `{code,message,details}`.
- Deterministic cursor pagination.
- Idempotency keys for dangerous writes.

## Resources
/currencies, /fx-rates, /accounts, /holdings, /positions, /transactions, /valuations, /ledger/accounts, /journal-entries, /reports/net-worth, /reports/cash-flow.

Holding endpoints must not imply one account; positions expose custody-specific state.

## Transaction taxonomy
PURCHASE, SALE, DEPOSIT, WITHDRAWAL, TRANSFER, DIVIDEND, INTEREST, FEE, EXPENSE, DEBT_DRAW, DEBT_PAYMENT, ADJUSTMENT. Keep taxonomy extensible.

## Journal API
Clients must never create a POSTED unbalanced journal entry.

## Reports
Expose missing valuation, missing FX, stale prices, partial data, and reconciliation gaps.

Domain changes come before persistence and OpenAPI changes. Breaking API changes require explicit migration decisions.
