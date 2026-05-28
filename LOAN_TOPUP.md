# Loan Product: Top-Up (LOAN_TOPUP)

**System**: Loan Settlement System (LSS)
**Event Code**: `LOAN_TOPUP`
**Status**: Draft
**Last Updated**: 2026-05-28

> For system overview, business rules, and shared concepts — see [LSS_Overview.md](LSS_Overview.md).

---

## What It Is

A top-up settlement closes an existing loan contract and opens a new loan account with a higher principal. The borrower may receive additional cash on top of the payoff amount.

**Settlement balance:**

```
New Principal = Closing Amount (B) + Cash-out (A) + Insurance (C) + Fees (D)
```

Supports both **NO_FTR** (cash) and **HAS_FTR** (bank transfer) disbursement flows.

---

## Flow: NO_FTR (Cash Disburse)

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
                  │
                  ├──[FAIL]──► Publish SettlementValidationFailed ──► Onigiri
                  │
                  ▼
            TOPUP_LOAN ───────────────────────────────────────────► Gringotts
                  │         (close previous loan + open new loan)
                  ├──[SUCCESS]──► Publish LoanSettled ──────────► Onigiri / DaVinci
                  │                       │
                  │               [Settlement complete]
                  │                       │
                  │               Continue fire-and-forget:
                  │               ├──► CREATE_SAP_CONTRACT          (optional — failure handled separately)
                  │               ├──► CLOSE_SAP_CONTRACT           (optional — failure handled separately)
                  │               ├──► CREATE_BOOKKEEPING_ACCOUNT   (optional — failure handled separately)
                  │               ├──► UPDATE_FACILITY              (optional, if HAS_FACILITY)
                  │               └──► SEND_INSURANCE_CONFIRM       (optional, if HAS_INSURANCE)
                  │
                  └──[FAIL after 3 retries]──► Publish SettlementFailed ──► Onigiri
```

---

## Flow: HAS_FTR (Transfer Disburse)

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
                  │
                  ├──[FAIL 400/500]──► Publish SettlementFailed ──► Onigiri
                  │
                  ▼
            Phase 3 (WORKER)
            ─────────────────
            VALIDATE_SETTLEMENT
                  │
                  ├──[FAIL]──► Publish SettlementValidationFailed ──► Onigiri
                  │
                  ▼
            TOPUP_LOAN ───────────────────────────────────────────► Gringotts
                  │
                  ├──[SUCCESS]──► Publish LoanSettled ──────────► Onigiri / DaVinci
                  │                       │
                  │               [Settlement complete]
                  │                       │
                  │               Continue fire-and-forget:
                  │               ├──► CREATE_SAP_CONTRACT          (optional — failure handled separately)
                  │               ├──► CLOSE_SAP_CONTRACT           (optional — failure handled separately)
                  │               ├──► CREATE_BOOKKEEPING_ACCOUNT   (optional — failure handled separately)
                  │               ├──► UPDATE_FACILITY              (optional, if HAS_FACILITY)
                  │               └──► SEND_INSURANCE_CONFIRM       (optional, if HAS_INSURANCE)
                  │
                  └──[FAIL after 3 retries]──► Publish SettlementFailed ──► Onigiri
```

---

## Phase/Step Configuration

### Phase 1 — API

| Phase Condition | Step | Action Code | Description |
|----------------|------|-------------|-------------|
| NO_FTR | 1 | `REGISTER_SETTLEMENT` | Persist settlement instruction. Structural balance check: Σ Fund Disburse = Σ Fund Usage. Assign Settlement ID. |
| HAS_FTR | 1 | `REGISTER_SETTLEMENT` | Same as above. |
| HAS_FTR | 2 | `REGISTER_FTR` | Forward Fund Transfer Request to Hora (mm-Hora). LSS enters pending state. |

### Phase 2 — MQ (HAS_FTR only)

| Phase Condition | Step | Action Code | Description |
|----------------|------|-------------|-------------|
| HAS_FTR | 1 | `REVIEW_FTR_RESPONSE` | Consume Hora's response from MQ. Fail settlement on HTTP 400/500. Proceed to Phase 3 on success. |

### Phase 3 — WORKER

| Step | Action Code | Condition | Blocks Settlement? | Description |
|------|-------------|-----------|-------------------|-------------|
| 1 | `VALIDATE_SETTLEMENT` | — | ✅ Yes | Fetch actual outstanding from Gringotts. Apply Rule 1, Rule 2, Rule 3. Reject or proceed. |
| 2 | `TOPUP_LOAN` | — | ✅ Yes | Call Gringotts Top-Up API to close the previous loan and open a new loan. **`LoanSettled` is published immediately on success.** |
| 3 | `CREATE_SAP_CONTRACT` | — | ❌ No | Open new loan contract in SAP-TR. Fire-and-forget. |
| 4 | `CLOSE_SAP_CONTRACT` | — | ❌ No | Mark previous loan contract as fully paid in SAP-TR. Fire-and-forget. Runs after `CREATE_SAP_CONTRACT`. |
| 5 | `CREATE_BOOKKEEPING_ACCOUNT` | — | ❌ No | Open new loan account in Bookkeeping. Fire-and-forget. |
| 6 | `SEND_INSURANCE_CONFIRM` | `HAS_INSURANCE` | ❌ No | Pay insurance premium to big mum (life) and/or Belly (non-life). Fire-and-forget. |
| 7 | `UPDATE_FACILITY` | `HAS_FACILITY` | ❌ No | Update facility in Ectoplasm. Fire-and-forget. |

---

## Success Definition

**Settlement is complete the moment `TOPUP_LOAN` succeeds in Gringotts.** LSS publishes `LoanSettled` immediately. All remaining steps are fire-and-forget.

| Step | Blocks Settlement? | On Failure |
|------|--------------------|------------|
| `VALIDATE_SETTLEMENT` | ✅ Yes | Settlement rejected; no execution occurs |
| `TOPUP_LOAN` | ✅ Yes | Retried up to 3 times; then `SettlementFailed` published |
| `CREATE_SAP_CONTRACT` | ❌ No | Investigated and fixed separately |
| `CLOSE_SAP_CONTRACT` | ❌ No | Investigated and fixed separately |
| `CREATE_BOOKKEEPING_ACCOUNT` | ❌ No | Investigated and fixed separately |
| `SEND_INSURANCE_CONFIRM` | ❌ No | Investigated and fixed separately |
| `UPDATE_FACILITY` | ❌ No | Investigated and fixed separately |

---

## Step Detail: VALIDATE_SETTLEMENT

### Purpose

Before execution, LSS fetches the **actual current outstanding** of the previous loan account from Gringotts. The borrower may have made a payment between when Onigiri submitted the instruction and when Phase 3 runs, changing the actual closing amount.

### Validation Rules

| Rule | Condition | Cause | Resolution |
|------|-----------|-------|------------|
| **Rule 1 — Shortage** | Σ Fund Disburse < Σ Fund Usage (actual) | Customer cancelled a receipt — actual closing amount is **greater** than application amount | **Reject.** Publish `SettlementValidationFailed` to Onigiri. No execution steps run. |
| **Rule 2 — Surplus** | Σ Fund Disburse > Σ Fund Usage (actual) | Customer made a repayment — actual closing amount is **less** than application amount | **Proceed.** LSS sends original payload to Gringotts unchanged. Gringotts posts reduced principal. `LoanSettled` published with adjustment details. |
| **Rule 3 — Cash-out floor** | Total Principal < Cash-out amount (Fund Usage A) | — | **Reject.** Publish `SettlementValidationFailed` to Onigiri. No execution steps run. |

### Rule 2 — LSS Behaviour

When Rule 2 is detected:
- LSS records the delta (agreed closing amount − actual closing amount)
- Proceeds to `TOPUP_LOAN` with the **original payload unchanged**
- Does **not** publish `SettlementAdjusted` before execution
- Does **not** redistribute the surplus to cash-out (Fund Usage A)

> Gringotts owns the recalculation. See TOPUP_LOAN step for Gringotts behaviour on Rule 2.

### Acceptance Criteria

**AC1 — Rule 1 reject: shortage (customer cancelled receipt)**

> Given the actual outstanding of the previous loan is **greater** than the application amount,
> when `VALIDATE_SETTLEMENT` runs,
> then LSS must reject the settlement, publish `SettlementValidationFailed` to Onigiri, and not proceed to `TOPUP_LOAN`.

**AC2 — Rule 2 detect: surplus (customer made repayment)**

> Given the actual outstanding of the previous loan is **less** than the application amount,
> when `VALIDATE_SETTLEMENT` runs,
> then LSS must detect the surplus, record the delta, and proceed to `TOPUP_LOAN` without modifying the payload.

**AC3 — Rule 3 reject: cash-out exceeds principal**

> Given the cash-out amount (Fund Usage A) is greater than the Total Principal,
> when `VALIDATE_SETTLEMENT` runs,
> then LSS must reject and publish `SettlementValidationFailed`.

**AC4 — Validation failure does not create or close any loan account**

> Given a settlement that fails `VALIDATE_SETTLEMENT`,
> then no Gringotts Top-Up API call must be made and no `LoanSettled` must be published.

---

## Step Detail: TOPUP_LOAN

### Purpose

LSS calls the **Gringotts Top-Up API** to close the previous loan account and open a new loan account in a single operation. LSS provides `previousLoanId`; Gringotts handles the close and open internally.

### Disbursement Composition

The new principal equals the closing amount of the previous loan plus any additional disbursement:

| Component | Fund Usage Type | Example (฿) |
|-----------|----------------|-------------|
| Previous loan closing amount (carry-over) | B — Loan Payoff | 200,000.00 |
| Net payment to borrower | A — Cash-out | 80,000.00 |
| Insurance premium | C — Insurance | 15,000.91 |
| Stamp duty | D — Fee | 1,000.00 |
| Rounding amount | — | 0.09 *(if applicable)* |
| **Total Principal Posted** | | **296,001.00** |

Rounding: if the raw sum has a fractional remainder, a rounding line item brings it to the nearest whole number.

### Interest Adjustment

If `loan.created_at` > `loan.effective_date`, Gringotts posts an interest adjustment automatically. LSS passes the disbursement date as `effective_date` in the request.

| Flow | Disbursement Date Source |
|------|-------------------------|
| `NO_FTR` | Provided by Onigiri in the settlement instruction |
| `HAS_FTR` | Provided by Hora in the MQ response (Phase 2) |

### Rule 2 — Reduced Principal (Gringotts Behaviour)

When LSS detected a Rule 2 surplus at `VALIDATE_SETTLEMENT`, Gringotts recalculates at execution time:

- Receives the original payload from LSS (unchanged)
- Fetches the actual closing amount of the previous loan
- Creates the new loan with **reduced principal** = actual closing amount + remaining components
- Returns the actual posted principal in the response

LSS then publishes `LoanSettled` with:
- Agreed principal (from the application)
- Actual principal posted (from Gringotts response)
- Delta amount

> The customer must be notified because the posted principal is less than agreed. Onigiri uses the adjustment details in `LoanSettled` to send notification (e.g. SMS).

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| Gringotts returns success (no adjustment) | LSS records new loan account ID. Publishes `LoanSettled` immediately. |
| Gringotts returns success (Rule 2 — reduced principal) | LSS reads actual posted principal. Publishes `LoanSettled` with adjustment details. |
| Gringotts returns error | LSS retries. Up to **3 attempts** total. |
| All 3 attempts fail | LSS publishes `SettlementFailed` to Onigiri. No further steps. |

### Acceptance Criteria

**AC1 — Previous loan ID is passed in the request**

> Given a Top-Up settlement,
> when `TOPUP_LOAN` executes,
> then the request must include `previousLoanId` so Gringotts can close it.

**AC2 — Accrued balance equals Total Principal (no Rule 2)**

> Given no Rule 2 surplus was detected,
> when `TOPUP_LOAN` executes successfully,
> then the new loan account must have an accrued balance equal to the agreed Total Principal.

**AC3 — Disbursement composition is posted**

> Given N disbursement components,
> when `TOPUP_LOAN` executes successfully,
> then the disbursement composition must contain exactly those N components, and the sum must equal the Total Principal.

**AC4 — Effective date equals disbursement date**

> Given a disbursement date of D,
> when `TOPUP_LOAN` executes successfully,
> then `loan.effective_date` must be D.

**AC5 — Interest adjustment posted when creation date is after effective date**

> Given `loan.created_at` > `loan.effective_date`,
> then Gringotts must post an interest adjustment for the period between the two dates.

**AC6 — Rule 2: Gringotts posts reduced principal**

> Given a Rule 2 surplus was detected,
> when `TOPUP_LOAN` executes,
> then Gringotts must fetch the actual closing amount and create the new loan with reduced principal = actual closing amount + remaining components.

**AC7 — Rule 2: actual posted principal is returned in the response**

> Given Gringotts applied a Rule 2 reduction,
> then the response must include the actual principal posted.

**AC8 — Rule 2: LoanSettled includes adjustment details**

> Given Rule 2 was detected and `TOPUP_LOAN` succeeds,
> when LSS publishes `LoanSettled`,
> then the event must include: agreed principal, actual principal posted, and delta amount.

**AC9 — Rule 2: cash-out amount is not increased**

> Given Rule 2 was detected,
> then the cash-out amount (Fund Usage A) must not be increased — the surplus reduces the principal only.

**AC10 — LoanSettled published immediately on success**

> Given `TOPUP_LOAN` returns success,
> then LSS must publish `LoanSettled` before proceeding to any fire-and-forget steps.

---

## Step Detail: CREATE_SAP_CONTRACT

> **Fire-and-forget** — runs after `LoanSettled` is published. Failure does not affect the settlement outcome.

### Purpose

LSS sends a single request to the **SAP Connector** to open a new loan contract in SAP-TR. This covers the **new contract only**. The previous contract is handled by `CLOSE_SAP_CONTRACT`.

### Transfer Channel

| Disbursement Flow | `houseBank` | `accountId` |
|-------------------|-------------|-------------|
| `NO_FTR` | `Z0000` | Business place code (5 digits) |
| `HAS_FTR` — KBANK | `K0593` | `C0701` |
| `HAS_FTR` — KKP Bank | `N0046` | `C1526` |

### Cashflow Entries (New Contract)

| SAP Flow | Description | Operation | Posting Date | Doc Date | `houseBank` | `accountId` | Condition |
|----------|-------------|-----------|-------------|----------|-------------|-------------|-----------|
| `A103` | Principal increase from restructuring (เงินต้นเพิ่มขึ้นจากการปรับโครงสร้างหนี้) | `−` | `loan.created_at` | `loan.effective_date` | Transfer channel | Transfer channel | Always |
| `A100` | Principal increase — additional disburse (เงินต้นเพิ่มขึ้น) | `−` | `loan.created_at` | `loan.effective_date` | Transfer channel | Transfer channel | Always |
| `A214` | Insurance premium received pending remittance (เงินรับค่าเบี้ยประกันภัยรอนำส่ง) | `+` | `loan.created_at` | `loan.effective_date` | Transfer channel | Transfer channel | Only if `HAS_INSURANCE` |
| `A211` | Stamp duty received pending remittance (เงินรับค่าอากรแสตมป์รอนำส่ง) | `+` | `loan.created_at` | `loan.effective_date` | Transfer channel | Transfer channel | Only if stamp duty in composition |
| `A508` | Rounding up (เงินปัดเศษขึ้น) | `+` | `loan.created_at` | `loan.effective_date` | Transfer channel | Transfer channel | Only if rounding in composition |
| `A350` | Interest income (รายได้ดอกเบี้ยรับ) | `+` | `loan.created_at` | `loan.effective_date` | `Z0000` | `ZC004` | Only if `loan.created_at` > `loan.effective_date` |

> `A103` = carry-over principal from the previous loan. `A100` = net new principal disbursed on top of the payoff. Both are always posted for a Top-Up contract.

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| SAP Connector returns `200` | Step marked complete. |
| SAP Connector returns error | Failure logged. Investigated and fixed separately. |

> Retry policy TBD.

### Acceptance Criteria

**AC1 — A103 and A100 always posted**

> Given any Top-Up settlement where `CREATE_SAP_CONTRACT` runs,
> then `A103` and `A100` must both be included.

**AC2 — A214 posted only when insurance exists**

> Given `HAS_INSURANCE`, then `A214` must be included. Without insurance, `A214` must not be included.

**AC3 — A211 posted only when stamp duty exists**

> Given stamp duty in the composition, then `A211` must be included.

**AC4 — A508 posted only when rounding exists**

> Given principal was rounded up, then `A508` must be included with the rounding amount.

**AC5 — A350 posted only when creation date is after effective date**

> Given `loan.created_at` > `loan.effective_date`, then `A350` must be included with `houseBank` = `Z0000` and `accountId` = `ZC004`.

**AC6 — Transfer channel correct per disbursement flow**

> Given `NO_FTR`, all applicable entries (except `A350`) must use `houseBank` = `Z0000` and the 5-digit business place code.
> Given `HAS_FTR` via KBANK, entries must use `houseBank` = `K0593`, `accountId` = `C0701`.
> Given `HAS_FTR` via KKP Bank, entries must use `houseBank` = `N0046`, `accountId` = `C1526`.

---

## Step Detail: CLOSE_SAP_CONTRACT

> **Fire-and-forget** — runs after `LoanSettled` is published. Runs **after** `CREATE_SAP_CONTRACT`. Failure does not affect the settlement outcome.

### Purpose

LSS calls the **SAP Connector fully-paid API** to mark the previous loan contract as fully paid in SAP-TR. LSS provides repayment details from the Gringotts repayment response; the SAP Connector posts the corresponding cashflow entries.

### Cashflow Entries (Previous Contract — Fully Paid)

Each entry is conditional — LSS includes only entries whose income code is present in the Gringotts repayment response.

| SAP Flow | Description | Operation | Posting Date | Doc Date | `houseBank` | `accountId` | Condition |
|----------|-------------|-----------|-------------|----------|-------------|-------------|-----------|
| `A104` | Clear AR principal for restructuring (ล้าง AR เงินต้นเพื่อปรับโครงสร้างหนี้) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A026` | Clear AR interest for restructuring (ล้าง AR ดอกเบี้ยเพื่อปรับโครงสร้างหนี้) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A218` | Guarantor suspense account (บัญชีพักเจ้าหนี้ค่าค้ำประกัน) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A506` | Rounding income — top-up (รายได้ส่วนเกินจากการปัดเศษ-top up) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A045` | Restructure accrued AR | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A517` | Penalty income — restructuring (รายได้ค่าปรับกรณีปรับโครงสร้างหนี้) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A518` | Collection fee income — restructuring (รายได้ค่าทวงถามกรณีปรับโครงสร้างหนี้) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |

**Notes:**
- Posting date = `loan.created_at`, Doc date = `loan.created_at`
- `houseBank` is always `Z0000` — no KBANK/KKP variant for fully-paid flows
- `accountId` = branch code from the receipt of the previous loan closure (from Gringotts repayment response)

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| SAP Connector returns `200` | Step marked complete. |
| SAP Connector returns error | Failure logged. Investigated and fixed separately. |

> Retry policy TBD.

### Acceptance Criteria

**AC1 — Only flows matching repayment income codes are posted**

> Given the Gringotts repayment response contains income codes X, Y, Z,
> then only SAP flow entries for X, Y, Z must be included. Entries for absent income codes must not be included.

**AC2 — Posting date and doc date are both loan.created_at**

> All cashflow entries must have posting date = `loan.created_at` and doc date = `loan.created_at`.

**AC3 — houseBank is always Z0000**

> Regardless of disbursement flow (NO_FTR or HAS_FTR), all entries must use `houseBank` = `Z0000`.

**AC4 — accountId comes from the receipt branch**

> `accountId` for all entries must be the 5-digit branch code from the receipt in the Gringotts repayment response.

**AC5 — CLOSE_SAP_CONTRACT runs after CREATE_SAP_CONTRACT**

> `CLOSE_SAP_CONTRACT` must not be called before `CREATE_SAP_CONTRACT` completes.

---

## Step Detail: CREATE_BOOKKEEPING_ACCOUNT

> **Fire-and-forget** — runs after `LoanSettled` is published. Failure does not affect the settlement outcome.

### Purpose

LSS calls the Bookkeeping API to open an accounting account for the new loan. Account opening only — no transactions posted.

### Request Fields

| Field | Source |
|-------|--------|
| Contract number | New loan account created by Gringotts (`TOPUP_LOAN`) |
| Branch code | Application record (via `application_uuid`) |
| Effective date | Disbursement date (`loan.effective_date`) |

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| Bookkeeping returns success | Step marked complete. |
| Bookkeeping returns error | Failure logged. Investigated and fixed separately. |

> Retry policy TBD.

### Acceptance Criteria

**AC1 — Account opened for new loan only**

> When `CREATE_BOOKKEEPING_ACCOUNT` runs,
> then the request must use the **new** loan contract number (not the previous loan), with branch code and effective date.

**AC2 — No transaction is posted**

> LSS must not post any cashflow or transaction entries — account opening only.

---

## Step Detail: SEND_INSURANCE_CONFIRM

> **Fire-and-forget** — runs after `LoanSettled` is published. Condition: `HAS_INSURANCE`.

### Purpose

LSS pays the insurance premium using the **order number** from the application record.

### Insurance Routing

| Insurance Type | System |
|---------------|--------|
| Life insurance | **big mum** |
| Non-life insurance (VMI / CMI) | **Belly** |

If both insurance types exist, LSS calls both systems independently. Each call is treated independently — one failure does not cancel the other.

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| Insurance API returns success | Confirmation reference stored. Step marked complete. |
| Insurance API returns error | Failure logged. Investigated and fixed separately. |

> Retry policy TBD.

### Acceptance Criteria

**AC1 — Routes to big mum for life insurance**

> Given a settlement with life insurance,
> when `SEND_INSURANCE_CONFIRM` runs,
> then LSS must call the big mum payment API with the order number from the application record.

**AC2 — Routes to Belly for non-life insurance**

> Given a settlement with non-life insurance (VMI or CMI),
> then LSS must call the Belly payment API with the order number from the application record.

**AC3 — Both systems called when settlement has both insurance types**

> Given both life and non-life insurance,
> then LSS must call big mum and Belly independently.

**AC4 — Confirmation reference is stored**

> Given an insurance API call returns success,
> then LSS must store the confirmation reference against the settlement record.

**AC5 — One insurance call failure does not block the other**

> Given both big mum and Belly are called and one fails,
> then the successful call is marked complete and the failed call is logged and marked failed independently.

---

## Step Detail: UPDATE_FACILITY

> **Fire-and-forget** — runs after `LoanSettled` is published. Condition: `HAS_FACILITY`.

### Purpose

LSS updates Ectoplasm to reflect the new loan account. Behaviour differs by facility type.

### Facility Types

| Facility Type | Description |
|--------------|-------------|
| **Single** | One loan per facility. Top-up creates a new facility (Onigiri). LSS closes the previous facility and links the new loan to the new facility. |
| **Revolving** | Multiple loans under the same facility. The facility is not closed — the previous loan is deactivated and the new loan is added to the same facility. |

LSS reads `facility_type` from the Ectoplasm facility record to decide which path to follow.

### Single Facility — Step Sequence

| Sub-step | Action | Target | Fields |
|----------|--------|--------|--------|
| 1 | PATCH previous facility — close | Previous facility ID | `status` = closed, `closing_date` = TBC |
| 2 | PATCH new facility — link loan | New facility ID | `loan_id` = new loan account ID |
| 3 | PATCH new facility — set previous | New facility ID | `previous_facility_id` = previous facility ID |

### Revolving Facility — Step Sequence

| Sub-step | Action | Target | Fields |
|----------|--------|--------|--------|
| 1 | PATCH previous loan in facility — deactivate | Previous loan entry | `is_active` = false |
| 2 | PATCH facility — add new loan | Same facility ID | Add new loan account ID |

No facility is closed. `previous_facility_id` is not set.

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| All Ectoplasm calls return success | Step marked complete. |
| Any Ectoplasm call returns error | Failure logged. Investigated and fixed separately. |

> Retry policy TBD.

### Acceptance Criteria

**AC1 — Facility type is read before taking action**

> When `UPDATE_FACILITY` runs,
> then LSS must read `facility_type` from the Ectoplasm facility record before deciding which path to follow.

**AC2 — Single: previous facility is closed**

> Given a single facility, LSS must PATCH the previous facility with status = closed and a closing date.

**AC3 — Single: new loan is linked to new facility**

> Given a single facility, LSS must PATCH the new facility with `loan_id` of the new Gringotts loan account.

**AC4 — Single: previous_facility_id is set on new facility**

> Given a single facility, LSS must PATCH the new facility with `previous_facility_id` = the ID of the previous facility.

**AC5 — Revolving: previous loan is deactivated in facility**

> Given a revolving facility, LSS must PATCH the previous loan entry with `is_active` = false.

**AC6 — Revolving: new loan is added to the same facility**

> Given a revolving facility, LSS must PATCH the existing facility to add the new loan account ID — the facility must not be closed.

**AC7 — Revolving: previous_facility_id is not set**

> Given a revolving facility, `previous_facility_id` must not be set and no facility must be closed.

**AC8 — Step skipped when no facility**

> Given a settlement without `HAS_FACILITY`,
> then `UPDATE_FACILITY` must not run and no call is made to Ectoplasm.
