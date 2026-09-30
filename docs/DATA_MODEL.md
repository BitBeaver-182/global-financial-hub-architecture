# Data Model

## Principles
- Separate custody, economic, and accounting concepts.
- Preserve historical facts.
- Avoid a polymorphic god table.
- Use PostgreSQL numeric/Decimal for money and quantities.
- Use extensible classifications.
- Enforce hard invariants in the database where practical.

## Core tables
users, currencies, fx_rates, accounts, holdings, positions, transactions, valuations, asset_liability_links, ledger_accounts, journal_entries, journal_lines.

Future type-specific tables may include security_details, real_estate_details, debt_details, vehicle_details, cash_details.

## Constraints
Target DB constraints include currency uniqueness, appropriate position/valuation uniqueness, positive quantities where required, and journal lines that contain exactly one of debit/credit. Posted entries must be balanced. Some cross-row rules require service transactions or triggers.

## Financial values
Money and quantities are Decimal/numeric. Native amount/currency remain authoritative; converted values are derived.

## Lifecycle
Prefer status/lifecycle fields over destructive deletion when history is affected.

Financially relevant records should carry stable IDs, timestamps, and source/posting/reversal metadata as applicable.
