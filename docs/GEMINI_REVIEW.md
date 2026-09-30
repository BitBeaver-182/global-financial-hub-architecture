# Gemini Conversation Review

This document records architectural observations extracted from the Gemini conversation. It is a review, not an approval of every proposal in that conversation.

## What we should carry into the product

### 1. Cash-flow visualization

A personal-finance dashboard benefits from a visual relationship between income, taxes/withholding, spending, savings, and investments. A Sankey-style cash-flow view is a useful candidate UI.

**Status:** Product candidate.

The backend should expose normalized cash-flow data; the frontend may render the Sankey with any appropriate visualization library. The chart library is not part of the financial domain model.

### 2. Spending + budget + forecast should be connected

Transactions are facts. Spending is an aggregation of facts. Budgets are plans/targets. Cash flow is movement over a period. Forecasts are projections.

These concepts should remain distinct in the domain, but the product should make their relationships visible instead of forcing users to navigate disconnected screens.

**Status:** Adopt as product design principle.

### 3. Country-based financial views

Useful candidate views:
- spending by country
- assets/exposure by country
- country-level account distribution

The backend should preserve country information on relevant counterparties, merchants, institutions, accounts, and/or assets where it is actually known. It should not infer a country merely from a currency.

**Status:** Product candidate + data-model consideration.

### 4. Yearly activity/spending heatmap

A GitHub-style calendar is a good candidate for showing daily spending intensity, transaction activity, or other daily metrics.

The backend should expose daily aggregates; the visualization is a frontend concern.

**Status:** Product candidate.

### 5. User categorization rules

The rule-engine discussion is valuable and should be planned.

Proposed pipeline:

raw external transaction
-> normalized transaction
-> system classification
-> user rule evaluation
-> user override
-> canonical transaction projection

Important: raw imported data must remain recoverable. Classification changes must not destroy the original source payload.

Rules should support deterministic precedence, conditions, and actions, with auditability.

**Status:** Architecture candidate; exact rule model is not yet an accepted ADR.

### 6. Opening/baseline onboarding

The idea of allowing a user to start from their current financial situation without reconstructing years of history is important.

A user can say:
- "I have 50 AAPL shares today."
- "My apartment is worth X today."
- "This bank account currently has Y."
- "I owe Z on this mortgage."

The system can establish an opening/baseline position without asking for historical source-of-funds details that are unavailable or irrelevant.

**Status:** Adopt as UX principle.

## Important corrections to the Gemini conversation

### Formance is an accounting/money-movement core, not the entire wealth model

Current Formance documentation describes Ledger as an account-based programmable ledger with atomic multi-posting transactions and arbitrary assets. Its asset notation does not provide a currency registry or economic taxonomy.

Therefore our application still needs its own:
- Currency metadata
- Holding/economic-object model
- Position model
- Valuation/price model
- Asset classifications
- Cost-basis/tax-lot policy
- Reporting/analytics model

Formance is the ledger engine. It is not the complete domain model for this product.

### Multi-currency does not mean FX conversion belongs in Formance

Formance can record different assets such as EUR, USD, BTC, or custom assets. Our app remains responsible for FX-rate history and reporting conversion policy.

Never assume that because a ledger accepts multiple assets it also knows how to value one asset in another.

### Net worth is not simply "the Formance balance"

For a wealth application, net worth can include:
- cash balances
- security positions multiplied by market prices
- manually valued real estate
- receivables
- liabilities

Therefore:

**Formance accounting truth** != **complete market-value net-worth report**

The report layer combines ledger balances, positions, valuations/prices, and FX according to explicit reporting rules.

### Holding must not be forced into one custody account

The original user requirement described a Holding that always belongs to one Account. That is convenient for a simple UI but creates a structural limitation.

A security can exist in multiple brokerage/custody accounts at the same time. The architecture therefore keeps:

Holding = economic identity
Position = quantity/balance in one custody Account

This distinction is intentional and should remain.

### Balance snapshots are observations, not replacements for financial history

Balance-only tracking is useful for:
- accounts that cannot provide detailed transactions
- historical onboarding
- reconciliation
- assets with periodic valuations

But a snapshot does not explain why a balance changed.

The system should therefore support both:
- event/transaction history
- point-in-time observations/snapshots

A reconciliation adjustment must be explicitly classified as an adjustment/reconciliation effect rather than silently becoming income or expense.

### Opening Balance Equity must not be confused with current market valuation

An opening balance can be a valid balancing mechanism when historical acquisition details are unavailable.

However, "current apartment value = 200,000" is a valuation fact, not automatically an acquisition-cost fact. We must keep:
- opening/baseline state
- acquisition information, when known
- market valuation

as separate concepts.

### Formance integration needs a consistency boundary

NestJS and Formance will be separate systems. We should not assume a normal HTTP request can atomically commit both PostgreSQL and Formance.

The production integration therefore needs:
- idempotent posting
- a durable intent/state record
- correlation to the Formance transaction ID
- retry handling
- reconciliation
- failure visibility

This is more important than adding more microservices early.

### Webhooks are not a safe foundation for the Community Edition assumption

Current Formance documentation places Webhooks in the Enterprise feature set. The architecture should not require Formance Webhooks for correctness.

For an open-source/self-hosted design, treat ledger log export/event streaming or controlled polling as the integration path unless the deployed edition explicitly provides webhooks.

### Docker is a development convenience, not automatically a production deployment decision

The Formance project provides an all-in-one Docker quick start, while its current production guidance points self-hosted users toward the Kubernetes operator. Therefore our architecture docs should distinguish local development from production deployment.

## Features to bring to the frontend early

These do not require committing the backend implementation yet:

- Cash-flow Sankey
- Spending-vs-budget progress/pacing
- Contextual transaction detail showing category/budget/cash-flow impact
- Country spending/assets map
- GitHub-style yearly activity heatmap
- Net-worth history plus forecast line
- Account/holding onboarding from current balances without historical source-of-funds friction

The frontend can prototype these with mock data while the backend contract is stabilized.

## Features to design in the backend before production

- Rule engine and merchant normalization
- Balance observations/snapshots
- Opening/baseline entries
- Valuations and historical prices
- FX-rate history and conversion policy
- Analytics/read-model contract
- Idempotent Formance posting boundary
- Import/source records and reconciliation
- Data-quality metadata for incomplete valuations/FX/reconciliation

## Explicitly not accepted yet

The Gemini conversation contains useful implementation suggestions that should not become architectural facts without review, including:
- exact charting-library choices
- exact Open Banking provider choices
- exact Redis TTLs
- a specific Vault/encryption library
- a specific analytics database
- claims about Formance capacity
- claims that any one framework/library eliminates accounting design work

These remain implementation options, not domain decisions.
