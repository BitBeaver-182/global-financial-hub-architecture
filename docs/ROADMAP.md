# Roadmap

## Phase 0 — Foundation
Repository, vocabulary, accounting invariant, Holding/Position/Account separation, currency model, API conventions, decision log.

## Phase 1 — Domain stabilization
Entity boundaries, accounting policies, transaction taxonomy, security/instrument model, property/debt model, lifecycle semantics.

## Phase 2 — Persistence contract
PostgreSQL/Prisma schema, constraints/indexes, atomic posting, audit/source metadata, import identifiers and reconciliation hooks.

## Phase 3 — API contract
OpenAPI, commands vs queries, errors, pagination/filtering, idempotency, report data-quality metadata.

## Phase 4 — Accounting engine
Posting, reversal, reconciliation, multi-currency, valuation, investment gains, debt schedules.

## Phase 5 — Production backend
Implement the production backend in a separate repository against versioned contracts.

The frontend integrates through the API contract rather than becoming the source of backend semantics.
