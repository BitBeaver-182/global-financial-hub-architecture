# Accounting Model

## Invariant
Every POSTED journal entry must satisfy total debits = total credits. Posting is atomic.

## Ledger types
ASSET, LIABILITY, EQUITY, INCOME, EXPENSE.

## Examples
**Apple purchase, 1,500 USD**
- Dr Investment Asset — Apple 1,500
- Cr Cash — Broker 1,500

**Salary, 5,000 USD**
- Dr Cash 5,000
- Cr Salary Income 5,000

**Cash transfer, 2,000 EUR**
- Dr Cash — Bank B 2,000
- Cr Cash — Bank A 2,000

**Property purchase, 200,000 EUR; 50,000 cash + 150,000 mortgage**
- Dr Real Estate Asset 200,000
- Cr Cash 50,000
- Cr Mortgage Liability 150,000

**Mortgage payment, 1,000 EUR; 800 principal + 200 interest**
- Dr Mortgage Liability 800
- Dr Interest Expense 200
- Cr Cash 1,000

Security transfers preserve economic identity and adjust positions; they are not sell+buy.

## Valuation
Market valuation is separate from acquisition cost. Unrealized-gain posting policy is OPEN and must not be assumed.

## Corrections
Posted history is append-oriented. Prefer reversals and correcting entries over silent mutation.

Conceptual states: DRAFT, POSTED, REVERSED, VOIDED.

Reconciliation matches external records to canonical history without destroying either source.
