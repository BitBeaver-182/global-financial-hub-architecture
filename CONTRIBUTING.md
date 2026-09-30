# Contributing

This is a shared design space for humans and AI agents.

## Before changing the model
Ask: what real-world concept is this; entity/event/relationship/snapshot/derived view; accounting meaning; ownership vs custody vs accounting; history requirements; duplicate sources of truth; invariants; migration risk.

## Required order
1. Decisions
2. Domain model
3. Accounting semantics
4. Data model
5. Prisma
6. API design
7. OpenAPI
8. Examples

Do not start from a frontend shape and work backwards into the domain.

## AI rules
- Read README and DECISIONS first.
- Preserve invariants.
- Separate facts, assumptions, and open questions.
- Do not invent business rules because the UI needs a field.
- Do not silently delete financial history.
- Use extensible data instead of closed UI enums where appropriate.
- Never silently default missing financial data, especially FX.
- Transfers are first-class semantics, not fake expense/income or sell/buy pairs.
- Keep custody and accounting separate.
- Use DB constraints for hard invariants and service validation for domain rules.
- If a proposal conflicts with an ADR, update the ADR explicitly.

The frontend is historical context and UX evidence, not the canonical backend schema.
