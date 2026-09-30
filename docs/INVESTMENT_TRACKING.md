# Investment Tracking / Portfolio Model

## Why this exists

The first useful investment feature does not need imported transaction history.

The app can start by tracking:
- securities/instruments
- positions
- accounts/custody
- market prices and valuations
- optional cost basis
- logical portfolios
- historical portfolio value/performance

This is compatible with a later transaction engine. User-facing transaction import is a later capability, not a prerequisite for the initial investment product.

## Proposed vocabulary

### Instrument
The market/security identity used by the pricing/reference-data layer.

Examples:
- Apple common stock
- VWCE ETF
- a government bond

Instrument metadata may include ticker, ISIN, exchange, country/region, asset class, and provider identifiers.

Instrument is not the user's ownership state.

### Holding
The user's economic ownership/obligation.

A holding can refer to an instrument, or represent a non-market asset such as real estate.

### Position
The quantity/balance of a holding in a custody Account.

Example:
- Holding: VWCE
- Account: DEGIRO
- Position: 120.5 shares

### Portfolio
A logical reporting/grouping concept.

A Portfolio is NOT a custody Account.

Examples:
- Long Term
- Retirement
- Speculative
- Family

A portfolio may group positions across several accounts.

## Important design constraint

A single custody position can potentially be split across logical portfolios.

Example:
- DEGIRO holds 100 VWCE shares.
- 60 shares are assigned to Long Term.
- 40 shares are assigned to Retirement.

Therefore, if portfolios need allocation at position level, use a membership/allocation model such as:

Portfolio
  -> PortfolioPosition
      -> Position
      -> allocatedQuantity

Do not put `portfolioId` directly on Position unless the product explicitly guarantees one portfolio per position.

## Initial investment workflow

The first version can support:

1. Create/select an Instrument.
2. Create/select a custody Account.
3. Create a Position with an as-of date and quantity.
4. Optionally record known acquisition cost/basis.
5. Record market valuations/prices over time.
6. Optionally assign part/all of the position to a Portfolio.
7. Calculate current value and historical performance from Position + Price/Valuation + FX.

The user does not need to import their brokerage's transaction history to use this workflow.

## Accounting relationship

The user-facing action "Add 50 shares of VWCE I own today" is not necessarily a user transaction.

Internally, the accounting system can establish the opening/baseline state through a ledger posting.

This keeps the ledger foundation without forcing transaction-entry UX.

## What transaction history unlocks later

Imported/manual transactions become important for:
- cost basis reconstruction
- tax lots
- realized gains
- dividends
- fees
- cash-flow analysis
- trade history
- performance attribution
- reconciliation

Those are later capabilities.

## Product sequencing proposal

### Investment MVP
Position + valuation + price history + FX + portfolio grouping.

### Investment transactions
Buy/sell/transfer/dividend/fees and cost basis/tax lots.

### Brokerage integrations
Sync current positions first; add transaction/activity history later when justified.

This sequencing lets the product deliver investment tracking and net worth early without making transaction ingestion the critical path.
