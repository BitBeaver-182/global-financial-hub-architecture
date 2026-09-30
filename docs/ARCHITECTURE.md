# Architecture

## Layers
1. **Economic:** holdings, positions, liabilities, valuations.
2. **Event:** purchases, sales, deposits, transfers, dividends, fees, debt events, adjustments.
3. **Accounting:** ledger accounts, journal entries, journal lines.

## Flow
Frontend -> OpenAPI -> Backend API -> Domain services -> Accounting/posting engine -> PostgreSQL

## Sources of truth
- Posted journal entries: accounting truth.
- Positions: units/balances held in custody.
- Valuations: observations, never replacements for acquisition facts.
- Reports: derived views and must expose important data-quality limitations.

## Custody
A holding can have many positions:
Holding Apple -> DEGIRO 40 shares; IBKR 10 shares.
A transfer changes custody without creating a fake sale.

## Liabilities
A property and its mortgage are distinct holdings with an explicit relationship.

## Integrations
External source -> raw import -> matching/reconciliation -> domain event -> posting -> position/valuation updates. External data must not blindly overwrite canonical history.

## Multi-currency
Keep native amounts. Reporting conversion uses explicit FX observations and a documented historical-rate policy.

Business logic is intentionally deferred until these semantics are stable.
