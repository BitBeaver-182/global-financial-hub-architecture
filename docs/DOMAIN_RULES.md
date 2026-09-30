# Domain Rules

## Ownership/custody
- Holding = economic object.
- Position = amount held in an account.
- One holding can have many positions.
- Transfer changes custody, not economic identity.
- Debt is a liability, not negative cash.

## Accounting
- Posted entries balance.
- Posting is atomic.
- Corrections do not silently mutate posted history.
- Transfers do not create fake income/expense.
- Principal repayment reduces liability; interest is expense.

## Valuation
- Acquisition cost is historical.
- Valuation is an observation.
- Missing current valuation is explicit.
- Never use acquisition price as implicit current value.

## Debt
Property and mortgage are distinct holdings linked explicitly.

## Currency
Native currency is preserved; conversion uses explicit FX; missing FX is unavailable.

## Forecasts
Forecasts are plans, not accounting truth. Generated rows must remain distinguishable from actual financial events.
