# Loan Settlement System — Requirements

**System**: Loan Settlement System (LSS)
**Caller**: Onigiri (Application System)
**Status**: Draft
**Last Updated**: 2026-05-25

---

## Business Function

Orchestrate the end-to-end financial settlement of a loan agreement after it has been signed between borrower and lender. A settlement is a double-entry financial transaction: the sum of all Fund Disburse (sources, `+`) must equal the sum of all Fund Usage (applications, `-`). Loan Settlement Engine validates this balance, finalises the execution plan, coordinates all sub-systems, and maintains an auditable settlement trail.

### Settlement Balance

| Fund Disburse — Source (`+`) | Fund Usage — Application (`-`) |
|------------------------------|-------------------------------|
| Principal outstanding posted to new loan account in Gringotts | **A** — Net cash payment to borrower (cash-out via Hora) |
| | **B** — Payoff of existing loan (closing previous loan in Gringotts) |
| | **C** — Insurance premium deduction (Life Insurance / VMI / CMI) |
| | **D** — Fee deductions (VAT, stamp duty, origination fee, legal fee) |

```
Principal = Cash-out (A) + Loan Payoff (B) + Insurance Premium (C) + Fees (D)
```

> Not all Fund Usage types apply to every settlement. A new loan with cash-out only uses A. A restructure uses B + A. A loan with insurance and fees uses A + C + D. Any combination is valid as long as both sides balance.

**Example — Top-Up with insurance and fees:**

| Side | Item | Amount (฿) |
|------|------|-----------|
| Fund Disburse (+) | Top-up principal posted to existing loan account | 350,000 |
| Fund Usage (−) | B: Payoff of existing loan outstanding | 200,000 |
| Fund Usage (−) | A: Net cash to borrower | 120,000 |
| Fund Usage (−) | C: Life insurance premium | 20,000 |
| Fund Usage (−) | D: Stamp duty | 10,000 |
| **Total** | | **350,000 = 350,000 ✓** |

---

## Table of Contents

1. [Overview](#1-overview)
2. [System Landscape](#2-system-landscape)
3. [Settlement Concepts](#3-settlement-concepts)
4. [Orchestration Model](#4-orchestration-model)
5. [Loan Product: New Book (LOAN_NEWBOOK)](#5-loan-product-new-book-loan_newbook)
6. [Loan Product: Top-Up (TBC)](#6-loan-product-top-up-tbc)
7. [Loan Product: Restructure (TBC)](#7-loan-product-restructure-tbc)
8. [Business Rules (Common)](#8-business-rules-common)
9. [Insurance Routing Logic (Common)](#9-insurance-routing-logic-common)
10. [Success and Failure Behaviour (Common)](#10-success-and-failure-behaviour-common)
11. [Events (Common)](#11-events-common)
12. [Open Questions](#12-open-questions)

---

## 1. Overview

LSS is triggered by Onigiri to finalise the financial settlement of a loan. Onigiri provides all monetary amounts (Fund Disburse and Fund Usage line items) in the settlement instruction. LSS validates those amounts against actual system state at execution time, then orchestrates all downstream systems to complete the settlement.

There are two entry cases depending on how the loan is disbursed:

### Case 1 — Cash Disburse (NO_FTR)

The borrower receives the loan proceeds as cash. **Disbursement has already occurred before Onigiri calls LSS.** Onigiri sends the settlement instruction with the disbursement details, and LSS proceeds directly to settlement execution.

**LSS responsibilities:**
1. Receive and register the settlement instruction from Onigiri
2. Validate fund balance and apply pre-execution business rules (Rule 1 / 2 / 3)
3. Orchestrate execution: create loan in Gringotts, register SAP contract, open Bookkeeping account, update facility, process insurance
4. Publish settlement outcome to Onigiri

### Case 2 — Transfer Disburse (HAS_FTR)

The borrower receives the loan proceeds via bank transfer. **Disbursement has not yet occurred when Onigiri calls LSS.** Onigiri sends the settlement instruction with the transfer details. LSS registers the instruction and forwards a Fund Transfer Request (FTR) to Hora (mm-Hora) to initiate the actual bank transfer. Once Hora completes the disbursement, it sends the transaction result back to LSS via MQ. LSS then proceeds to settlement execution.

**LSS responsibilities:**
1. Receive and register the settlement instruction from Onigiri
2. Forward a Fund Transfer Request (FTR) to Hora (mm-Hora) to initiate the bank transfer
3. Receive disbursement confirmation from Hora via MQ after the transfer is executed
4. Validate fund balance and apply pre-execution business rules (Rule 1 / 2 / 3)
5. Orchestrate execution: create loan in Gringotts, register SAP contract, open Bookkeeping account, update facility, process insurance
6. Publish settlement outcome to Onigiri

---

## 2. System Landscape

| System | Description |
|--------|-------------|
| **Onigiri** | Application system. Triggers LSS via API. Subscribes to settlement outcome events. |
| **Gringotts** | Core Bank. Loan account creation, top-up, and closure. |
| **SAP (SAP-TR)** | Contract registration system. Records loan contract per settlement. |
| **Bookkeeping** | Accounting system. Opens accounting entry per settlement. |
| **Ectoplasm** | Facility Management System. Links loan accounts to credit facilities. |
| **Hora (mm-Hora)** | Host-to-Host (H2H) fund disbursement system. Used for transfer flows. |
| **big mum** | Life insurance service. Receives insurance confirmation for life insurance premium. |
| **Belly** | Motor insurance service. Receives insurance confirmation for VMI and CMI. |
| **DaVinci** | Downstream system. Subscribes to `LoanSettled` event. |

---

## 3. Settlement Concepts

### Fund Disburse vs Fund Usage

A settlement is a double-entry financial transaction. Every baht disbursed must be accounted for by a corresponding usage:

```
Σ (Fund Disburse line items) = Σ (Fund Usage line items)
```

| Side | Direction | Examples |
|------|-----------|---------|
| **Fund Disburse** | Source (+) | New loan principal posted to Gringotts |
| **Fund Usage** | Application (−) | Cash-out to borrower, loan payoff, insurance premium, fees |

---

## 4. Orchestration Model

### Phase/Step Configuration

Settlement execution is driven by a configuration table stored in the database. This allows the sequence of operations to vary per loan product type and disbursement method without code changes.

**Config table structure:**

| Column | Description |
|--------|-------------|
| `event_type_id` | Links to the loan event type (e.g. New Book, Top-Up, Restructure) |
| `event_code` | Human-readable event code (e.g. `LOAN_NEWBOOK`) |
| `phase_order` | Execution phase: 1 = API, 2 = MQ, 3 = WORKER |
| `trigger_type` | How the phase is triggered: `API`, `MQ`, `WORKER` |
| `phase_condition` | Optional condition that gates the entire phase (e.g. `HAS_FTR`, `NO_FTR`) |
| `step_order` | Execution order of the action within the phase |
| `action_code` | Identifier for the operation to execute |
| `action_description` | Human-readable description |
| `action_condition` | Optional condition that gates the individual step (e.g. `HAS_FACILITY`, `HAS_INSURANCE`) |

### Phase Summary

| Phase | Trigger | When It Runs |
|-------|---------|-------------|
| 1 | API | Immediately on Onigiri's API call to LSS |
| 2 | MQ | Only for HAS_FTR — when Hora publishes disbursement result to MQ |
| 3 | WORKER | After Phase 1 (NO_FTR) or Phase 2 (HAS_FTR) completes |

---

## 5. Loan Product: New Book (LOAN_NEWBOOK)

A new book settlement creates a brand-new loan account in Gringotts.

### 5.1 Flow: Cash Disburse (NO_FTR)

**Trigger**: Onigiri calls LSS after cash is physically lent to the borrower.

```
Onigiri ──API──► LSS
                  │
                  ▼
            Phase 1 (API, NO_FTR)
            ──────────────────────
            REGISTER_SETTLEMENT
                  │
                  ▼
            Phase 3 (WORKER)
            ─────────────────
            VALIDATE_SETTLEMENT
            CREATE_LOAN
            CREATE_SAP_CONTRACT
            CREATE_BOOKKEEPING_ACCOUNT
            UPDATE_FACILITY        (if HAS_FACILITY)
            SEND_INSURANCE_CONFIRM (if HAS_INSURANCE)
                  │
                  ▼
            Publish settlement event ──► Onigiri / DaVinci
```

### 5.2 Flow: Transfer Disburse (HAS_FTR)

**Trigger**: Onigiri calls LSS after a transfer disbursement request is reviewed and approved. Actual disbursement has not yet occurred — LSS coordinates it via Hora.

```
Onigiri ──API──► LSS
                  │
                  ▼
            Phase 1 (API, HAS_FTR)
            ───────────────────────
            REGISTER_SETTLEMENT
            REGISTER_FTR ──────────────────────► Hora (mm-Hora)
                  │                                      │
            [LSS pending — awaiting H2H result]          │
                  │                                      │
                  ▼                                      ▼
            Phase 2 (MQ, HAS_FTR) ◄──── Hora publishes result to MQ
            ──────────────────────
            REVIEW_FTR_RESPONSE
            (fail on 400/500)
                  │
                  ▼
            Phase 3 (WORKER)
            ─────────────────
            VALIDATE_SETTLEMENT
            CREATE_LOAN
            CREATE_SAP_CONTRACT
            CREATE_BOOKKEEPING_ACCOUNT
            UPDATE_FACILITY        (if HAS_FACILITY)
            SEND_INSURANCE_CONFIRM (if HAS_INSURANCE)
                  │
                  ▼
            Publish settlement event ──► Onigiri / DaVinci
```

### 5.3 Phase/Step Configuration (LOAN_NEWBOOK)

#### Phase 1 — API

| Phase Condition | Step | Action Code | Description |
|----------------|------|-------------|-------------|
| NO_FTR | 1 | `REGISTER_SETTLEMENT` | Persist settlement instruction. Structural balance check: Σ Fund Disburse = Σ Fund Usage. Assign Settlement ID. |
| HAS_FTR | 1 | `REGISTER_SETTLEMENT` | Same as above. |
| HAS_FTR | 2 | `REGISTER_FTR` | Forward Fund Transfer Request to Hora (mm-Hora). LSS enters pending state. |

#### Phase 2 — MQ (HAS_FTR only)

| Phase Condition | Step | Action Code | Description |
|----------------|------|-------------|-------------|
| HAS_FTR | 1 | `REVIEW_FTR_RESPONSE` | Consume Hora's response from MQ. Fail settlement on HTTP 400/500. Proceed to Phase 3 on success. |

#### Phase 3 — WORKER

| Step | Action Code | Condition | Description |
|------|-------------|-----------|-------------|
| 1 | `VALIDATE_SETTLEMENT` | — | Fetch actual outstanding from Gringotts. Apply Rule 1, Rule 2, Rule 3. Reject or adjust before proceeding. |
| 2 | `CREATE_LOAN` | — | Create new loan account in Gringotts. **Settlement success event is published after this step.** |
| 3 | `CREATE_SAP_CONTRACT` | — | Register loan contract in SAP-TR. Non-blocking. |
| 4 | `CREATE_BOOKKEEPING_ACCOUNT` | — | Open accounting entry in Bookkeeping. Non-blocking. |
| 5 | `UPDATE_FACILITY` | `HAS_FACILITY` | Link the new loan account to the existing facility in Ectoplasm. Non-blocking. |
| 6 | `SEND_INSURANCE_CONFIRM` | `HAS_INSURANCE` | Send insurance confirmation to the appropriate service. Routes based on insurance type (see [Section 9](#9-insurance-routing-logic-common)). Non-blocking. |

### 5.4 Facility Update — New Book

Applies when `HAS_FACILITY` condition is met.

**Action**: Link the newly created Gringotts loan account to the existing facility record in Ectoplasm.

No previous facility is closed. `previous_facility_id` is not set.

---

## 6. Loan Product: Top-Up *(TBC)*

> **Status**: Step configuration to be confirmed. Known differences from New Book are documented below.

A top-up settlement adds funds to an existing loan account in Gringotts rather than creating a new one.

### 6.1 Known Step Differences from New Book

| Area | New Book | Top-Up |
|------|----------|--------|
| Gringotts action | `CREATE_LOAN` (new account) | TBC — expected to be a top-up/update on existing account |
| SAP action | `CREATE_SAP_CONTRACT` | TBC |
| Bookkeeping action | `CREATE_BOOKKEEPING_ACCOUNT` | TBC |
| Facility update | Link new account to facility | Close previous facility → link new account to new facility → set `previous_facility_id` |

### 6.2 Facility Update — Top-Up

Applies when `HAS_FACILITY` condition is met.

**Sequence**:
1. Close the previous facility in Ectoplasm.
2. Add the new (top-up) loan account to the new facility.
3. Set `previous_facility_id` on the new facility record to reference the closed facility.

**Facility record structure (relevant fields):**

| Field | Description |
|-------|-------------|
| `id` | Facility ID |
| `facility_owner_type_id` | Owner type |
| `facility_owner_number` | Owner reference number |
| `facility_type_id` | Facility type |
| `previous_facility_id` | Reference to the prior facility (set during top-up) |

### 6.3 Full Step Configuration

> TBC — full phase/step config table to be defined once confirmed with engineering.

---

## 7. Loan Product: Restructure *(TBC)*

> **Status**: Step configuration to be confirmed. Known differences are documented below.

A restructure closes an existing loan and opens a new one under revised terms.

### 7.1 Known Step Differences from New Book

| Area | New Book | Restructure |
|------|----------|-------------|
| Gringotts action | `CREATE_LOAN` (new account) | TBC — expected to include closing old account + creating new account |
| Facility update | Link new account to facility | Close previous facility → link new account to new facility → set `previous_facility_id` |

### 7.2 Facility Update — Restructure

Same logic as Top-Up (see [Section 6.2](#62-facility-update--top-up)).

### 7.3 Full Step Configuration

> TBC — full phase/step config table to be defined once confirmed with engineering.

---

## 8. Business Rules (Common)

These rules apply across all loan product types.

### Structural Balance Check (Phase 1 — REGISTER_SETTLEMENT)

Performed immediately on receipt of the Onigiri instruction using the amounts provided by Onigiri:

```
Σ (Fund Disburse line items) = Σ (Fund Usage line items)
```

If this check fails, the settlement is rejected at Phase 1 before any record is persisted.

### Pre-Execution Rules (Phase 3 — VALIDATE_SETTLEMENT)

These rules require fetching the **actual current outstanding** from Gringotts at execution time, because account balances may have changed between when Onigiri sent the instruction and when execution begins (e.g. borrower made an early payment).

| Rule | Condition | Resolution |
|------|-----------|------------|
| **Rule 1 — Shortage** | Σ Fund Disburse < Σ Fund Usage (actual) | **Reject.** Publish `SettlementValidationFailed` to Onigiri with: shortage amount, affected line item(s), reason. No execution occurs. |
| **Rule 2 — Surplus** | Σ Fund Disburse > Σ Fund Usage (actual) | **Proceed with adjustment.** Recalculate affected Fund Usage line items. Surplus is redistributed to cash-out (Fund Usage A). Publish `SettlementAdjusted` to Onigiri with delta details before executing. |
| **Rule 3 — Cash-out floor** | Principal disbursed < Cash-out amount | **Reject.** Lender cannot disburse more cash to borrower than the principal posted to the loan account. |

---

## 9. Insurance Routing Logic (Common)

Applies when `HAS_INSURANCE` condition is met on the `SEND_INSURANCE_CONFIRM` step. LSS calls a different insurance service endpoint based on the insurance product type attached to the loan.

| Insurance Type | Service | Endpoint |
|---------------|---------|----------|
| Life Insurance Premium | **big mum** | big mum API |
| Voluntary Motor Insurance (VMI) Premium | **Belly** | Belly API |
| Compulsory Motor Insurance (CMI) Premium | **Belly** | Belly API |

> **Open question**: Whether Life Insurance and Motor Insurance are configured as two separate `action_code` entries, or as a single `SEND_INSURANCE_CONFIRM` action that branches internally to the correct endpoint, is TBD (see [Section 12](#12-open-questions)).

---

## 10. Success and Failure Behaviour (Common)

### Settlement Success

Settlement is considered **successful and published to Onigiri/DaVinci** immediately after `CREATE_LOAN` in Gringotts succeeds. All subsequent Phase 3 steps (SAP, Bookkeeping, Facility, Insurance) are non-blocking — their outcomes do not affect the settlement success publication.

### Non-Blocking Step Failure and Retry

For each non-blocking Phase 3 step (CREATE_SAP_CONTRACT, CREATE_BOOKKEEPING_ACCOUNT, UPDATE_FACILITY, SEND_INSURANCE_CONFIRM):

1. On failure, LSS retries the step automatically up to **3 attempts**.
2. If all 3 attempts fail, LSS publishes a failure event to Onigiri.

### Phase 2 Failure (HAS_FTR only)

If `REVIEW_FTR_RESPONSE` receives a 400 or 500 response from Hora:
- Settlement is marked as failed.
- Failure event is published to Onigiri.
- Phase 3 does not execute.

### Validation Failure (Rule 1 or Rule 3)

- Settlement is rejected.
- `SettlementValidationFailed` is published to Onigiri.
- No execution steps run.

---

## 11. Events (Common)

| Event | Published By | Subscribed By | Trigger |
|-------|-------------|---------------|---------|
| `LoanSettled` | LSS | Onigiri, DaVinci | `CREATE_LOAN` succeeds in Gringotts |
| `SettlementValidationFailed` | LSS | Onigiri | Rule 1 or Rule 3 rejection in `VALIDATE_SETTLEMENT` |
| `SettlementAdjusted` | LSS | Onigiri | Rule 2 surplus adjustment before execution |
| `SettlementFailed` | LSS | Onigiri | Phase 2 FTR failure, or non-blocking step exhausts 3 retries |

---

## 12. Open Questions

| # | Question | Impact |
|---|----------|--------|
| 1 | Is `SEND_INSURANCE_CONFIRM` one `action_code` that branches internally, or two separate `action_code` entries (one per insurance service) in the config table? | Affects step config design and how insurance type is determined at runtime |
| 2 | Is `VALIDATE_SETTLEMENT` an explicit `action_code` entry in the Phase 3 config, or an implicit pre-check that always runs before Phase 3 steps begin? | Affects whether validation can be disabled or reordered per loan type |
| 3 | Is `SettlementAdjusted` (Rule 2) a blocking notification — must Onigiri acknowledge before execution proceeds — or non-blocking? | Affects execution sequencing and latency |
| 4 | What is the SAP-TR decommission timeline and when does the parallel dual-write period start? | Scoping for dual-write transition work |
| 5 | What is the full phase/step configuration for Top-Up and Restructure event types? | Required to complete Sections 6 and 7 |
| 6 | For Rule 2 surplus redistribution — is the default always to redirect surplus to cash-out (Fund Usage A), or can Onigiri specify an alternative destination? | Affects adjustment logic in `VALIDATE_SETTLEMENT` |
