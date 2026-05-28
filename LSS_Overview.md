# Loan Settlement System — Overview

**System**: Loan Settlement System (LSS)
**Status**: Draft
**Last Updated**: 2026-05-27

---

## What LSS Does

LSS orchestrates the end-to-end financial settlement of a loan after a loan agreement is confirmed by Onigiri (the application system). It validates fund balances, then executes a configurable sequence of steps across multiple downstream systems to open the loan, register accounting records, and disburse or apply funds.

---

## Systems

| System | Role |
|--------|------|
| **Onigiri** | Application system — triggers LSS, subscribes to settlement events |
| **Gringotts** | Core Bank — loan account creation and management |
| **SAP (SAP-TR)** | Contract registration |
| **Bookkeeping** | Accounting system |
| **Ectoplasm** | Facility Management System |
| **Hora (mm-Hora)** | Host-to-Host disbursement system for fund transfer |
| **big mum** | Life insurance service |
| **Belly** | Motor insurance service (Voluntary and Compulsory) |
| **DaVinci** | Downstream subscriber to `LoanSettled` event |

---

## Settlement Balance

A settlement is a double-entry financial transaction. Every baht disbursed must be accounted for by a corresponding usage:

```
Σ (Fund Disburse line items) = Σ (Fund Usage line items)
```

| Fund Disburse — Source (+) | Fund Usage — Application (−) |
|----------------------------|------------------------------|
| Principal posted to loan account in Gringotts | **A** — Cash-out to borrower (via Hora) |
| | **B** — Payoff of existing loan |
| | **C** — Insurance premium |
| | **D** — Fees (stamp duty, VAT, etc.) |

Not all Fund Usage types apply to every product. Any combination is valid as long as both sides balance.

---

## Disbursement Flows

| Flow | Condition | Description |
|------|-----------|-------------|
| **NO_FTR** | Borrower receives cash in hand | Disbursement has already occurred. Onigiri calls LSS directly. LSS proceeds to execution immediately. |
| **HAS_FTR** | Borrower receives funds via bank transfer | Disbursement has not yet occurred. LSS forwards a Fund Transfer Request (FTR) to Hora and waits for the disbursement confirmation via MQ before proceeding to execution. |

---

## Orchestration Model

Settlement execution is driven by a **phase/step configuration table in the database**. Each loan product type has its own set of phases and steps. Steps can be conditional (e.g. only run if the loan has a facility, or only run if the loan has insurance).

| Phase | Trigger | When It Runs |
|-------|---------|-------------|
| **1 — API** | Onigiri API call | Immediately on receipt of the settlement instruction |
| **2 — MQ** | Hora MQ message | HAS_FTR only — after Hora confirms the fund transfer |
| **3 — WORKER** | Internal | After Phase 1 (NO_FTR) or Phase 2 (HAS_FTR) completes |

**Settlement is complete the moment the core loan step (e.g. `CREATE_LOAN`, `TOPUP_LOAN`, `RESTRUCTURE_LOAN`) succeeds in Gringotts.** LSS publishes `LoanSettled` immediately at that point. All remaining Phase 3 steps run as **fire-and-forget** — their outcome does not affect the settlement result.

---

## Business Rules

### Structural Balance Check (Phase 1 — `REGISTER_SETTLEMENT`)

Performed immediately on receipt using amounts provided by Onigiri:

```
Σ (Fund Disburse) = Σ (Fund Usage)
```

If this fails, the settlement is rejected at Phase 1 before any record is persisted.

### Pre-Execution Validation (Phase 3 — `VALIDATE_SETTLEMENT`)

LSS fetches the **actual current outstanding** from Gringotts at execution time, because account balances may have changed since Onigiri submitted the instruction.

| Rule | Condition | Resolution |
|------|-----------|------------|
| **Rule 1 — Shortage** | Σ Fund Disburse < Σ Fund Usage (actual) | **Reject.** Publish `SettlementValidationFailed` to Onigiri. No execution occurs. |
| **Rule 2 — Surplus** | Σ Fund Disburse > Σ Fund Usage (actual) | **Proceed.** LSS passes original payload to Gringotts. Gringotts posts reduced principal. `LoanSettled` published with adjustment details. |
| **Rule 3 — Cash-out floor** | Principal < Cash-out amount | **Reject.** Publish `SettlementValidationFailed`. Applies only when Fund Usage A exists. |

> Rule 2 behaviour differs slightly by product — see individual product specs for details.

---

## Events

| Event | Trigger |
|-------|---------|
| `LoanSettled` | Core loan step succeeds in Gringotts. Published to Onigiri and DaVinci. |
| `SettlementValidationFailed` | Rule 1 or Rule 3 rejection. Published to Onigiri. |
| `SettlementFailed` | Phase 2 FTR failure, or core loan step fails after 3 retries. Published to Onigiri. |

---

## Insurance Routing

Applies when the `HAS_INSURANCE` condition is met on `SEND_INSURANCE_CONFIRM`.

| Insurance Type | System |
|---------------|--------|
| Life insurance | **big mum** |
| Voluntary Motor Insurance (VMI) | **Belly** |
| Compulsory Motor Insurance (CMI) | **Belly** |

If a settlement has both life and non-life insurance, LSS calls both systems independently.

---

## Open Questions

| # | Question | Impact |
|---|----------|--------|
| 1 | Is `SEND_INSURANCE_CONFIRM` one `action_code` that branches internally, or two separate entries (one per insurance service)? | Step config design |
| 2 | Is `VALIDATE_SETTLEMENT` an explicit `action_code` in Phase 3 config, or an implicit pre-check? | Whether validation can be disabled per loan type |
| 3 | Is Rule 2 `SettlementAdjusted` a blocking notification (Onigiri must acknowledge) or non-blocking? | Execution sequencing |
| 4 | What is the SAP-TR decommission timeline and when does the dual-write period start? | Dual-write transition scoping |
| 5 | For Rule 2 surplus in New Book — is the surplus always redistributed to cash-out (Fund Usage A), or can Onigiri specify an alternative destination? | Adjustment logic |
