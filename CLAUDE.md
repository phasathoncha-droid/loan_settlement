# Loan Settlement System (LSS)

## What It Is

LSS is a backend orchestration service that finalises the financial settlement of a loan after a loan agreement is confirmed by Onigiri (the application system). It validates fund balances, then executes a configurable sequence of steps across multiple downstream systems to open the loan, register accounting records, and disburse or apply funds.

## Two Entry Flows

| Flow | Condition | Description |
|------|-----------|-------------|
| **Cash (NO_FTR)** | Borrower receives cash in hand | Onigiri calls LSS directly after cash is lent. LSS registers the settlement and proceeds to execution immediately. |
| **Transfer (HAS_FTR)** | Borrower receives funds via bank transfer | Onigiri calls LSS, which registers the settlement and forwards a Fund Transfer Request (FTR) to Hora (H2H disbursement system). LSS waits for the disbursement confirmation via MQ before proceeding to execution. |

## Key Systems

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

## Orchestration Model

Settlement execution is driven by a **phase/step configuration stored in the database**. Each settlement event type (e.g. `LOAN_NEWBOOK`) has phases with ordered steps. Steps can be conditional (e.g. only run if the loan has a facility, or only run if the loan has insurance). The sequence can differ per loan type.

Phases: `1 = API` → `2 = MQ (transfer only)` → `3 = WORKER`

Settlement is published as **successful once CREATE_LOAN in Gringotts succeeds**. All subsequent Phase 3 steps are non-blocking — failures are retried up to 3 times independently and do not affect the success publication.

## Requirements Document

See [Loan_Settlement_Requirements.md](Loan_Settlement_Requirements.md) for full flow details, business rules, action specifications, and integration contracts.
