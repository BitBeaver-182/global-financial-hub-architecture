# Architecture Decisions

## Accepted

- ADR-001: Separate backend architecture repository.
- ADR-002: Accounting-first foundation.
- ADR-003: Holding and Position are separate.
- ADR-004: Account and LedgerAccount are separate.
- ADR-005: Money and quantities use Decimal/numeric.
- ADR-006: Currencies are extensible data.
- ADR-007: Native currency is preserved.
- ADR-008: Missing FX is explicit, never silently 1.
- ADR-009: API ranges use from inclusive / to exclusive.
- ADR-010: Categories are data, not UI-only enums.
- ADR-011: Valuation is separate from acquisition cost.
- ADR-012: Transfers are first-class semantics.
- ADR-013: Posted history is append-oriented; corrections use reversals/adjustments.
- ADR-014: Frontend is not backend authority.
- ADR-015: Architecture before production implementation.

## Open decisions

The following require explicit decisions before production implementation:

### Accounting/domain
- chart of accounts
- tax lots and cost basis
- realized/unrealized gain treatment
- fee capitalization
- valuation accounting
- trade date vs settlement date
- portfolio hierarchy
- corporate actions
- lot tracking
- pending vs settled transactions
- treatment of opening/baseline balances
- treatment of balance reconciliation adjustments

### Integration and infrastructure
- import and reconciliation strategy
- recurring transaction semantics
- authentication and tenant model
- audit log
- period close/lock
- Formance/app consistency boundary and durable posting intents
- source identifiers and idempotency strategy
- event/log export consumption strategy
- production Formance deployment model

### Analytics/product
- analytics read model strategy
- historical snapshot cadence and authority
- forecast semantics and confidence/data-quality metadata
- rule engine schema and deterministic precedence
- merchant normalization strategy
- country attribution rules
- cache strategy and invalidation/versioning

Open decisions must be resolved through an explicit ADR, with alternatives and consequences documented.
