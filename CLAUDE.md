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

Settlement execution is driven by a **phase/step configuration stored in the database**. Each settlement event type has phases with ordered steps. Steps can be conditional (e.g. only run if the loan has a facility, or only run if the loan has insurance).

Phases: `1 = API` → `2 = MQ (transfer only)` → `3 = WORKER`

Settlement is **complete the moment the core Gringotts step succeeds** — `LoanSettled` is published to Onigiri and DaVinci immediately. LSS then continues the remaining Phase 3 steps as **fire-and-forget background operations**.

## Loan Products

| Product | Event Code | File |
|---------|-----------|------|
| New Book | `LOAN_NEWBOOK` | [LOAN_NEWBOOK.md](LOAN_NEWBOOK.md) |
| Top-Up | `LOAN_TOPUP` | [LOAN_TOPUP.md](LOAN_TOPUP.md) |
| Restructure | `LOAN_RESTRUCTURE` | [LOAN_RESTRUCTURE.md](LOAN_RESTRUCTURE.md) |

## Requirements Documents

| Document | Description |
|----------|-------------|
| [Loan_Settlement_Requirements.md](Loan_Settlement_Requirements.md) | Index — links to all product specs and explains how to add a new loan product |
| [LSS_Overview.md](LSS_Overview.md) | System overview — rules, events, orchestration model |
| [LOAN_NEWBOOK.md](LOAN_NEWBOOK.md) | Full spec for New Book |
| [LOAN_TOPUP.md](LOAN_TOPUP.md) | Full spec for Top-Up |
| [LOAN_RESTRUCTURE.md](LOAN_RESTRUCTURE.md) | Full spec for Restructure |

## Adding a New Loan Product

When a new product is added:
1. Create `LOAN_<EVENTCODE>.md` following the structure of existing product files
2. Add a row to the Loan Product Summary table in `Loan_Settlement_Requirements.md`
3. Add a row to the Loan Products table above in this file
