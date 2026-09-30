# Domain Model

## Core entities
- **User:** owner/tenant of financial data; auth model remains open.
- **Currency:** stable ID, code, name, symbol, scale.
- **Account:** custody/location such as bank, broker, wallet.
- **Holding:** economic thing owned or owed.
- **Position:** quantity/balance of a holding in one account.
- **Transaction:** domain financial event.
- **Valuation:** dated observation of value.
- **LedgerAccount:** accounting bucket.
- **JournalEntry:** balanced accounting event.
- **JournalLine:** debit/credit line.
- **AssetLiabilityLink:** explicit relation such as property -> mortgage.

Holding and Position are separate: Holding 1:N Position N:1 Account.

Account and LedgerAccount are separate concepts.

Transaction and JournalEntry are separate: the transaction says what happened; journal entries represent its accounting effect.

## Taxonomy
Top level: INVESTMENT, CASH, OTHER_ASSET, LIABILITY.

Investment: EQUITY, ETF, CRYPTO, REAL_ESTATE, BOND, COMMODITY, PRIVATE_EQUITY, OTHER.

Liability: MORTGAGE, LOAN, CREDIT_CARD, STUDENT_LOAN, OTHER.

Use extensible classification data where growth is expected.

## Value
Current value is derived from applicable valuation/price and position, with explicit data availability. Never silently fall back to acquisition cost.
