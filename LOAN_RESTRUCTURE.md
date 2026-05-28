# Loan Product: Restructure (LOAN_RESTRUCTURE)

**System**: Loan Settlement System (LSS)
**Event Code**: `LOAN_RESTRUCTURE`
**Status**: Draft
**Last Updated**: 2026-05-27

> For system overview, business rules, and shared concepts — see [LSS_Overview.md](LSS_Overview.md).

---

## What It Is

A restructure closes an existing loan contract and opens a new loan contract under revised payment terms (e.g. expanded tenor, reduced instalment amount). No net new cash is disbursed to the borrower — the new principal equals the total closing amount of the previous loan.

**Settlement balance:**

```
New Principal = Previous Principal (carry-over) + Accrued Balance (interest + fees)
```

| Fund Disburse (+) | Fund Usage (−) |
|-------------------|----------------|
| New principal posted to new loan account | B — Payoff / closing of previous loan |

There is no Fund Usage A (cash-out). Always **NO_FTR** — no bank transfer required.

---

## Flow: NO_FTR (only flow for Restructure)

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
            RESTRUCTURE_LOAN ─────────────────────────────────────► Gringotts
                  │
                  ├──[SUCCESS]──► Publish LoanSettled ──────────► Onigiri / DaVinci
                  │                       │
                  │               [Settlement complete]
                  │                       │
                  │               Continue fire-and-forget:
                  │               ├──► CREATE_SAP_CONTRACT          (optional — failure handled separately)
                  │               ├──► CLOSE_SAP_CONTRACT           (optional — failure handled separately)
                  │               ├──► CREATE_BOOKKEEPING_ACCOUNT   (optional — failure handled separately)
                  │               └──► UPDATE_FACILITY              (optional, if HAS_FACILITY)
                  │
                  └──[FAIL after 3 retries]──► Publish SettlementFailed ──► Onigiri
```

---

## Phase/Step Configuration

### Phase 1 — API

| Phase Condition | Step | Action Code | Description |
|----------------|------|-------------|-------------|
| NO_FTR | 1 | `REGISTER_SETTLEMENT` | Persist settlement instruction. Structural balance check: Σ Fund Disburse = Σ Fund Usage. Assign Settlement ID. |

> Restructure is always NO_FTR. Phase 2 (MQ) does not apply.

### Phase 3 — WORKER

| Step | Action Code | Condition | Blocks Settlement? | Description |
|------|-------------|-----------|-------------------|-------------|
| 1 | `VALIDATE_SETTLEMENT` | — | ✅ Yes | Fetch actual outstanding from Gringotts. Apply Rule 1, Rule 2. Reject or proceed. |
| 2 | `RESTRUCTURE_LOAN` | — | ✅ Yes | Call Gringotts Restructure API to close the previous loan and open a new loan. **`LoanSettled` is published immediately on success.** |
| 3 | `CREATE_SAP_CONTRACT` | — | ❌ No | Open new loan contract in SAP-TR. Fire-and-forget. |
| 4 | `CLOSE_SAP_CONTRACT` | — | ❌ No | Mark previous loan contract as fully paid in SAP-TR. Fire-and-forget. Runs after `CREATE_SAP_CONTRACT`. |
| 5 | `CREATE_BOOKKEEPING_ACCOUNT` | — | ❌ No | Open new loan account in Bookkeeping. Fire-and-forget. |
| 6 | `UPDATE_FACILITY` | `HAS_FACILITY` | ❌ No | Update facility in Ectoplasm. Fire-and-forget. |

> There is no `SEND_INSURANCE_CONFIRM` step — insurance is not included in a restructure settlement.

---

## Success Definition

**Settlement is complete the moment `RESTRUCTURE_LOAN` succeeds in Gringotts.** LSS publishes `LoanSettled` immediately. All remaining steps are fire-and-forget.

| Step | Blocks Settlement? | On Failure |
|------|--------------------|------------|
| `VALIDATE_SETTLEMENT` | ✅ Yes | Settlement rejected; no execution occurs |
| `RESTRUCTURE_LOAN` | ✅ Yes | Retried up to 3 times; then `SettlementFailed` published |
| `CREATE_SAP_CONTRACT` | ❌ No | Investigated and fixed separately |
| `CLOSE_SAP_CONTRACT` | ❌ No | Investigated and fixed separately |
| `CREATE_BOOKKEEPING_ACCOUNT` | ❌ No | Investigated and fixed separately |
| `UPDATE_FACILITY` | ❌ No | Investigated and fixed separately |

---

## Step Detail: VALIDATE_SETTLEMENT

### Purpose

Before execution, LSS fetches the **actual current outstanding** of the previous loan account from Gringotts. The borrower may have made a payment during the retention process, changing the actual closing amount.

### Validation Rules

| Rule | Condition | Cause | Resolution |
|------|-----------|-------|------------|
| **Rule 1 — Shortage** | Σ Fund Disburse < Σ Fund Usage (actual) | Customer cancelled a receipt — actual closing amount is **greater** than application amount | **Reject.** Publish `SettlementValidationFailed` to Onigiri. No execution steps run. |
| **Rule 2 — Surplus** | Σ Fund Disburse > Σ Fund Usage (actual) | Customer made a repayment during retention — actual closing amount is **less** than application amount | **Proceed.** LSS sends original payload to Gringotts unchanged. Gringotts posts reduced principal. `LoanSettled` published with adjustment details. |

> Rule 3 (cash-out floor) does not apply — there is no cash-out (Fund Usage A) in Restructure.

### Rule 2 — LSS Behaviour

When Rule 2 is detected:
- LSS records the delta (agreed closing amount − actual closing amount)
- Proceeds to `RESTRUCTURE_LOAN` with the **original payload unchanged**
- Does **not** publish `SettlementAdjusted` before execution

> Gringotts owns the recalculation. See RESTRUCTURE_LOAN step for Gringotts behaviour on Rule 2.

### Acceptance Criteria

**AC1 — Rule 1 reject: shortage (customer cancelled receipt)**

> Given the actual outstanding of the previous loan is **greater** than the application amount,
> when `VALIDATE_SETTLEMENT` runs,
> then LSS must reject the settlement, publish `SettlementValidationFailed` to Onigiri, and not proceed to `RESTRUCTURE_LOAN`.

**AC2 — Rule 2 detect: surplus (customer made repayment during retention)**

> Given the actual outstanding of the previous loan is **less** than the application amount,
> when `VALIDATE_SETTLEMENT` runs,
> then LSS must detect the surplus, record the delta, and proceed to `RESTRUCTURE_LOAN` without modifying the payload.

**AC3 — Validation failure does not create or close any loan account**

> Given a settlement that fails `VALIDATE_SETTLEMENT`,
> then no Gringotts Restructure API call must be made and no `LoanSettled` must be published.

---

## Step Detail: RESTRUCTURE_LOAN

### Purpose

LSS calls the **Gringotts Restructure API** to close the previous loan account and open a new loan account under revised terms in a single operation. LSS provides the `previousLoanId`, new principal, accrued balance breakdown, and new loan terms.

### Disbursement Composition

The new principal equals the closing amount of the previous loan — no additional cash is added:

| Component | Payload Field | Example (฿) |
|-----------|--------------|-------------|
| Previous principal (carry-over) | `previousPrincipal` | 18,127.00 |
| Previous accrued fees (interest + guarantee) | `previousFee` | 200.00 |
| **New Principal** | `principal` | 18,327.00 |

`principal` = `previousPrincipal` + `previousFee`

The `deducted` array lists items deducted from the new principal (e.g. stamp duty). These are already reflected in the principal — they are not added on top.

### Interest Adjustment

If `createdAt` > `loanEffectiveDate`, Gringotts posts an interest adjustment automatically. Since Restructure is always NO_FTR, the disbursement date is provided by Onigiri in the settlement instruction.

### Rule 2 — Reduced Principal (Gringotts Behaviour)

When LSS detected a Rule 2 surplus at `VALIDATE_SETTLEMENT`, Gringotts recalculates at execution time:

- Receives the original payload from LSS (unchanged)
- Fetches the actual closing amount of the previous loan
- Creates the new loan with **reduced principal** = actual closing amount at execution time
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

> Given a Restructure settlement,
> when `RESTRUCTURE_LOAN` executes,
> then the request must include `previousLoanId` so Gringotts can close it.

**AC2 — Principal equals previousPrincipal + previousFee**

> Given no Rule 2 adjustment,
> when `RESTRUCTURE_LOAN` executes successfully,
> then `principal` in the request must equal `previousPrincipal` + `previousFee`, and the new loan account must post that amount.

**AC3 — Effective date equals disbursement date**

> Given a disbursement date of D,
> when `RESTRUCTURE_LOAN` executes successfully,
> then `loanEffectiveDate` must be D.

**AC4 — Interest adjustment posted when creation date is after effective date**

> Given `createdAt` > `loanEffectiveDate`,
> then Gringotts must post an interest adjustment for the period between the two dates.

**AC5 — Rule 2: Gringotts posts reduced principal**

> Given a Rule 2 surplus was detected,
> when `RESTRUCTURE_LOAN` executes,
> then Gringotts must fetch the actual closing amount and create the new loan with reduced principal = actual closing amount at execution time.

**AC6 — Rule 2: actual posted principal is returned in the response**

> Given Gringotts applied a Rule 2 reduction,
> then the response must include the actual principal posted.

**AC7 — Rule 2: LoanSettled includes adjustment details**

> Given Rule 2 was detected and `RESTRUCTURE_LOAN` succeeds,
> when LSS publishes `LoanSettled`,
> then the event must include: agreed principal, actual principal posted, and delta amount.

**AC8 — LoanSettled published immediately on success**

> Given `RESTRUCTURE_LOAN` returns success,
> then LSS must publish `LoanSettled` before proceeding to any fire-and-forget steps.

---

## Step Detail: CREATE_SAP_CONTRACT

> **Fire-and-forget** — runs after `LoanSettled` is published. Failure does not affect the settlement outcome.

### Purpose

LSS sends a single request to the **SAP Connector** to open a new loan contract in SAP-TR. This covers the **new contract only**. The previous contract is handled by `CLOSE_SAP_CONTRACT`.

### Transfer Channel

Restructure is always NO_FTR. Channel is always BRANCH.

| `houseBank` | `accountId` |
|-------------|-------------|
| `Z0000` | Business place code (5 digits) |

### Cashflow Entries (New Contract)

| SAP Flow | Description | Operation | Posting Date | Doc Date | `houseBank` | `accountId` | Condition |
|----------|-------------|-----------|-------------|----------|-------------|-------------|-----------|
| `A103` | Principal increase from restructuring (เงินต้นเพิ่มขึ้นจากการปรับโครงสร้างหนี้) | `−` | `loan.created_at` | `loan.effective_date` | `Z0000` | Business place (5 digits) | Always |
| `A997` | AR interest from restructure (ดอกเบี้ยและค่าธรรมเนียมค้างชำระจากการปรับโครงสร้างหนี้) | `−` | `loan.created_at` | `loan.effective_date` | `Z0000` | Business place (5 digits) | Always |
| `A350` | Interest income (รายได้ดอกเบี้ยรับ) | `+` | `loan.created_at` | `loan.effective_date` | `Z0000` | `ZC004` | Only if `loan.created_at` > `loan.effective_date` |

> - `A103` = carry-over principal (`previousPrincipal`)
> - `A997` = total accrued fees from the previous loan (`previousFee` — interest + guarantee). Together A103 + A997 cover the full closing amount.
> - No `A100` (no net new principal), no `A214` (no insurance), no `A211` (no stamp duty), no `A508` (no rounding) for Restructure.

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| SAP Connector returns `200` | Step marked complete. |
| SAP Connector returns error | Failure logged. Investigated and fixed separately. |

> Retry policy TBD.

### Acceptance Criteria

**AC1 — A103 and A997 always posted**

> Given any Restructure settlement where `CREATE_SAP_CONTRACT` runs,
> then `A103` and `A997` must both be included.

**AC2 — A100, A214, A211, A508 are never posted for Restructure**

> Given any Restructure settlement,
> then `A100`, `A214`, `A211`, and `A508` must not be included.

**AC3 — A350 posted only when creation date is after effective date**

> Given `loan.created_at` > `loan.effective_date`, then `A350` must be included with `houseBank` = `Z0000` and `accountId` = `ZC004`.
> Given `loan.created_at` = `loan.effective_date`, then `A350` must not be included.

**AC4 — Transfer channel is always BRANCH (Z0000)**

> All entries must use `houseBank` = `Z0000` and the 5-digit business place code.

---

## Step Detail: CLOSE_SAP_CONTRACT

> **Fire-and-forget** — runs after `LoanSettled` is published. Runs **after** `CREATE_SAP_CONTRACT`. Failure does not affect the settlement outcome.

### Purpose

LSS calls the **SAP Connector fully-paid API** to mark the previous loan contract as fully paid in SAP-TR.

### Cashflow Entries (Previous Contract — Fully Paid)

Each entry is conditional — LSS includes only entries whose income code is present in the Gringotts repayment response.

| SAP Flow | Description | Operation | Posting Date | Doc Date | `houseBank` | `accountId` | Condition |
|----------|-------------|-----------|-------------|----------|-------------|-------------|-----------|
| `A104` | Clear AR principal for restructuring (ล้าง AR เงินต้นเพื่อปรับโครงสร้างหนี้) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A026` | Clear AR interest for restructuring (ล้าง AR ดอกเบี้ยเพื่อปรับโครงสร้างหนี้) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A218` | Guarantor suspense account (บัญชีพักเจ้าหนี้ค่าค้ำประกัน) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A506` | Rounding income — restructuring (รายได้ส่วนเกินจากการปัดเศษ) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A045` | Restructure accrued AR | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A517` | Penalty income — restructuring (รายได้ค่าปรับกรณีปรับโครงสร้างหนี้) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |
| `A518` | Collection fee income — restructuring (รายได้ค่าทวงถามกรณีปรับโครงสร้างหนี้) | `+` | `loan.created_at` | `loan.created_at` | `Z0000` | Branch from receipt (5-digit) | If income code present in repayment |

**Notes:**
- Posting date = `loan.created_at`, Doc date = `loan.created_at`
- `houseBank` is always `Z0000`
- `accountId` = branch code from the receipt of the previous loan closure

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

> All entries must use `houseBank` = `Z0000`.

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
| Contract number | New loan account created by Gringotts (`RESTRUCTURE_LOAN`) |
| Branch code | Application record (via `application_uuid`) |
| Effective date | Disbursement date (`loanEffectiveDate`) |

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

## Step Detail: UPDATE_FACILITY

> **Fire-and-forget** — runs after `LoanSettled` is published. Condition: `HAS_FACILITY`.

### Purpose

LSS updates Ectoplasm to reflect the new loan account created by the restructure. Behaviour differs by facility type — identical logic to Top-Up.

### Facility Types

| Facility Type | Description |
|--------------|-------------|
| **Single** | One loan per facility. Restructure creates a new facility (Onigiri). LSS closes the previous facility and links the new loan to the new facility. |
| **Revolving** | Multiple loans under the same facility. The facility is not closed — the previous loan is deactivated and the new loan is added. |

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
