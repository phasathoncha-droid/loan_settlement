# Loan Product: New Book (LOAN_NEWBOOK)

**System**: Loan Settlement System (LSS)
**Event Code**: `LOAN_NEWBOOK`
**Status**: Draft
**Last Updated**: 2026-05-27

> For system overview, business rules, and shared concepts — see [LSS_Overview.md](LSS_Overview.md).

---

## What It Is

A new book settlement creates a brand-new loan account in Gringotts. The borrower receives the full loan principal as cash or via bank transfer.

**Settlement balance:**

```
New Principal = Cash-out (A) + Insurance (C) + Fees (D)
```

Supports both **NO_FTR** (cash) and **HAS_FTR** (bank transfer) disbursement flows.

---

## Flow: NO_FTR (Cash Disburse)

Disbursement has already occurred before Onigiri calls LSS.

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
            CREATE_LOAN ──────────────────────────────────────────► Gringotts
                  │
                  ├──[SUCCESS]──► Publish LoanSettled ──────────► Onigiri / DaVinci
                  │                       │
                  │               [Settlement complete]
                  │                       │
                  │               Continue fire-and-forget:
                  │               ├──► CREATE_SAP_CONTRACT          (optional — failure handled separately)
                  │               ├──► CREATE_BOOKKEEPING_ACCOUNT   (optional — failure handled separately)
                  │               ├──► UPDATE_FACILITY              (optional, if HAS_FACILITY)
                  │               └──► SEND_INSURANCE_CONFIRM       (optional, if HAS_INSURANCE)
                  │
                  └──[FAIL after 3 retries]──► Publish SettlementFailed ──► Onigiri
```

---

## Flow: HAS_FTR (Transfer Disburse)

Disbursement has not yet occurred. LSS coordinates it via Hora.

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
            CREATE_LOAN ──────────────────────────────────────────► Gringotts
                  │
                  ├──[SUCCESS]──► Publish LoanSettled ──────────► Onigiri / DaVinci
                  │                       │
                  │               [Settlement complete]
                  │                       │
                  │               Continue fire-and-forget:
                  │               ├──► CREATE_SAP_CONTRACT          (optional — failure handled separately)
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
| 1 | `VALIDATE_SETTLEMENT` | — | ✅ Yes | Fetch actual outstanding from Gringotts. Apply Rule 1, Rule 2, Rule 3. Reject or adjust before proceeding. |
| 2 | `CREATE_LOAN` | — | ✅ Yes | Create new loan account in Gringotts. **`LoanSettled` is published immediately on success.** |
| 3 | `CREATE_SAP_CONTRACT` | — | ❌ No | Open loan contract in SAP-TR. Fire-and-forget. |
| 4 | `CREATE_BOOKKEEPING_ACCOUNT` | — | ❌ No | Open loan account in Bookkeeping. Fire-and-forget. |
| 5 | `SEND_INSURANCE_CONFIRM` | `HAS_INSURANCE` | ❌ No | Pay insurance premium to big mum (life) and/or Belly (non-life). Fire-and-forget. |
| 6 | `UPDATE_FACILITY` | `HAS_FACILITY` | ❌ No | Link new loan account to facility in Ectoplasm. Fire-and-forget. |

---

## Success Definition

**Settlement is complete the moment `CREATE_LOAN` succeeds in Gringotts.** LSS publishes `LoanSettled` to Onigiri and DaVinci immediately. All remaining steps are fire-and-forget and do not affect the settlement outcome.

| Step | Blocks Settlement? | On Failure |
|------|--------------------|------------|
| `VALIDATE_SETTLEMENT` | ✅ Yes | Settlement rejected; no execution occurs |
| `CREATE_LOAN` | ✅ Yes | Retried up to 3 times; then `SettlementFailed` published |
| `CREATE_SAP_CONTRACT` | ❌ No | Investigated and fixed separately |
| `CREATE_BOOKKEEPING_ACCOUNT` | ❌ No | Investigated and fixed separately |
| `SEND_INSURANCE_CONFIRM` | ❌ No | Investigated and fixed separately |
| `UPDATE_FACILITY` | ❌ No | Investigated and fixed separately |

---

## Step Detail: VALIDATE_SETTLEMENT

### Purpose

Before execution, LSS re-validates the settlement amounts using **actual current outstanding** from Gringotts. For a standard New Book, there is no existing loan to fetch — only Rule 3 applies.

### Validation Rules

| Rule | Condition | Resolution |
|------|-----------|------------|
| **Rule 3 — Cash-out floor** | Cash-out (Fund Usage A) > Total Principal | **Reject.** Publish `SettlementValidationFailed` to Onigiri. No execution occurs. |

> Rule 1 and Rule 2 are defined in [LSS_Overview.md](LSS_Overview.md). They do not apply to a standard New Book (no existing loan account to compare against).

### Acceptance Criteria

**AC1 — Rule 3 reject: cash-out exceeds principal**

> Given a New Book settlement where the cash-out amount (Fund Usage A) is greater than the Total Principal,
> when `VALIDATE_SETTLEMENT` runs,
> then LSS must reject the settlement, publish `SettlementValidationFailed` to Onigiri with the reason, and not proceed to `CREATE_LOAN`.

**AC2 — Rule 3 pass: cash-out does not exceed principal**

> Given a New Book settlement where the cash-out amount (Fund Usage A) is less than or equal to the Total Principal,
> when `VALIDATE_SETTLEMENT` runs,
> then LSS must pass validation and proceed to `CREATE_LOAN`.

**AC3 — Validation failure does not create a loan account**

> Given a settlement that fails `VALIDATE_SETTLEMENT`,
> then no loan account must be created in Gringotts and no `LoanSettled` event must be published.

---

## Step Detail: CREATE_LOAN

### Purpose

LSS instructs Gringotts to open a new loan account and post the **Total Principal** as the opening accrued balance.

### Disbursement Composition

The principal is broken down across each fund usage component:

| Component | Fund Usage Type | Example (฿) |
|-----------|----------------|-------------|
| Net payment to borrower | A — Cash-out | 10,000.00 |
| Insurance premium | C — Insurance | 2,000.91 |
| Stamp duty | D — Fee | 50.00 |
| Rounding amount | — | 0.09 *(if applicable)* |
| **Total Principal Posted** | | **12,051.00** |

When the raw sum has a fractional remainder, a rounding line item is added to bring it to the nearest whole number. If already a whole number, no rounding line item is included.

### Interest Adjustment

If `loan.created_at` > `loan.effective_date`, Gringotts posts an interest adjustment automatically.

| Flow | Disbursement Date Source |
|------|-------------------------|
| `NO_FTR` | Provided by Onigiri in the settlement instruction |
| `HAS_FTR` | Provided by Hora in the MQ response (Phase 2) |

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| Gringotts returns success | LSS records the new loan account ID. Publishes `LoanSettled` immediately. Proceeds to fire-and-forget steps. |
| Gringotts returns error | LSS retries automatically. Up to **3 attempts** total. |
| All 3 attempts fail | LSS publishes `SettlementFailed` to Onigiri. No further steps execute. |

### Acceptance Criteria

**AC1 — Accrued balance equals Total Principal**

> Given a New Book settlement with a Total Principal of ฿X,
> when `CREATE_LOAN` executes successfully,
> then the new loan account in Gringotts must have an accrued balance of exactly ฿X.

**AC2 — Disbursement composition is posted**

> Given a settlement with N disbursement components,
> when `CREATE_LOAN` executes successfully,
> then the disbursement composition on the new loan account must contain exactly those N components with the correct amounts, and the sum must equal the Total Principal.

**AC3 — Loan creation date is today**

> When `CREATE_LOAN` executes,
> then `loan.created_at` must be the current system date at time of execution.

**AC4 — Effective date equals disbursement date**

> Given a settlement where the disbursement date is D,
> when `CREATE_LOAN` executes successfully,
> then `loan.effective_date` must be D, regardless of the loan creation date.

**AC5 — Interest adjustment posted when creation date is after effective date**

> Given `loan.created_at` > `loan.effective_date`,
> when `CREATE_LOAN` executes successfully,
> then Gringotts must post an interest adjustment for the period from `loan.effective_date` to `loan.created_at`.

**AC6 — No interest adjustment when dates are equal**

> Given `loan.created_at` = `loan.effective_date`,
> then no interest adjustment is posted.

---

## Step Detail: CREATE_SAP_CONTRACT

> **Fire-and-forget** — runs after `LoanSettled` is published. Failure does not affect the settlement outcome.

### Purpose

LSS sends a single request to the **SAP Connector** to open a loan contract in SAP-TR. The SAP Connector internally executes: Create Person (if absent) → Create Contract → Post Cashflow. A `200` response means all three succeeded.

### Transfer Channel

| Disbursement Flow | `houseBank` | `accountId` |
|-------------------|-------------|-------------|
| `NO_FTR` | `Z0000` | Business place code (5 digits) |
| `HAS_FTR` — KBANK | `K0593` | `C0701` |
| `HAS_FTR` — KKP Bank | `N0046` | `C1526` |

### Cashflow Entries

| SAP Flow | Description | Operation | Posting Date | Doc Date | `houseBank` | `accountId` | Condition |
|----------|-------------|-----------|-------------|----------|-------------|-------------|-----------|
| `A100` | Principal increase (เงินต้นเพิ่มขึ้น) | `−` | `loan.created_at` | `loan.effective_date` | Transfer channel | Transfer channel | Always |
| `A214` | Insurance premium received pending remittance (เงินรับค่าเบี้ยประกันภัยรอนำส่ง) | `+` | `loan.created_at` | `loan.effective_date` | Transfer channel | Transfer channel | Only if `HAS_INSURANCE` |
| `A211` | Stamp duty received pending remittance (เงินรับค่าอากรแสตมป์รอนำส่ง) | `+` | `loan.created_at` | `loan.effective_date` | Transfer channel | Transfer channel | Only if stamp duty in composition |
| `A508` | Rounding up (เงินปัดเศษขึ้น) | `+` | `loan.created_at` | `loan.effective_date` | Transfer channel | Transfer channel | Only if rounding in composition |
| `A350` | Interest income (รายได้ดอกเบี้ยรับ) | `+` | `loan.created_at` | `loan.effective_date` | `Z0000` | `ZC004` | Only if `loan.created_at` > `loan.effective_date` |

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| SAP Connector returns `200` | Step marked complete. |
| SAP Connector returns error | Failure logged. Step marked failed. Investigated and fixed separately. |

> Retry policy TBD.

### Acceptance Criteria

**AC1 — A100 always posted**

> Given any New Book settlement where `CREATE_SAP_CONTRACT` runs,
> when the SAP Connector is called,
> then `A100` must be included with posting date = `loan.created_at` and doc date = `loan.effective_date`.

**AC2 — A214 posted only when insurance exists**

> Given a settlement with `HAS_INSURANCE`, then `A214` must be included.
> Given a settlement without insurance, then `A214` must not be included.

**AC3 — A211 posted only when stamp duty exists**

> Given a settlement with stamp duty in the composition, then `A211` must be included.
> Given no stamp duty, then `A211` must not be included.

**AC4 — A508 posted only when rounding exists**

> Given a settlement where principal was rounded up, then `A508` must be included with the rounding amount.
> Given no rounding, then `A508` must not be included.

**AC5 — A350 posted only when creation date is after effective date**

> Given `loan.created_at` > `loan.effective_date`, then `A350` must be included with `houseBank` = `Z0000` and `accountId` = `ZC004`.
> Given `loan.created_at` = `loan.effective_date`, then `A350` must not be included.

**AC6 — Transfer channel: BRANCH for NO_FTR**

> Given `NO_FTR`, all applicable entries must use `houseBank` = `Z0000` and the 5-digit business place code.

**AC7 — Transfer channel: KBANK for HAS_FTR via KBANK**

> Given `HAS_FTR` via KBANK, all applicable entries (except `A350`) must use `houseBank` = `K0593` and `accountId` = `C0701`.

**AC8 — Transfer channel: KKP for HAS_FTR via KKP Bank**

> Given `HAS_FTR` via KKP Bank, all applicable entries (except `A350`) must use `houseBank` = `N0046` and `accountId` = `C1526`.

---

## Step Detail: CREATE_BOOKKEEPING_ACCOUNT

> **Fire-and-forget** — runs after `LoanSettled` is published. Failure does not affect the settlement outcome.

### Purpose

LSS calls the Bookkeeping API to open an accounting account for the new loan. Account opening only — no transactions or cashflow entries are posted.

### Request Fields

| Field | Source |
|-------|--------|
| Contract number | Loan account created by Gringotts (`CREATE_LOAN`) |
| Branch code | Application record (via `application_uuid`) |
| Effective date | Disbursement date (`loan.effective_date`) |

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| Bookkeeping returns success | Step marked complete. |
| Bookkeeping returns error | Failure logged. Step marked failed. Investigated and fixed separately. |

> Retry policy TBD.

### Acceptance Criteria

**AC1 — Account is opened with correct fields**

> Given a New Book settlement where `CREATE_BOOKKEEPING_ACCOUNT` runs,
> when the Bookkeeping API is called,
> then the request must include the contract number, branch code, and effective date.

**AC2 — No transaction is posted**

> When the Bookkeeping API is called,
> then LSS must not post any cashflow or transaction entries — account opening only.

---

## Step Detail: SEND_INSURANCE_CONFIRM

> **Fire-and-forget** — runs after `LoanSettled` is published. Condition: `HAS_INSURANCE`.

### Purpose

LSS pays the insurance premium by calling the insurance system API using the **order number** from the application record.

### Insurance Routing

| Insurance Type | System |
|---------------|--------|
| Life insurance | **big mum** |
| Non-life insurance (VMI / CMI) | **Belly** |

If a settlement has both life and non-life insurance, LSS calls both systems independently.

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| Insurance API returns success | Confirmation reference stored. Step marked complete. |
| Insurance API returns error | Failure logged. Step marked failed. Investigated and fixed separately. |

> Each insurance call is treated independently — one failure does not cancel the other. Retry policy TBD.

### Acceptance Criteria

**AC1 — Routes to big mum for life insurance**

> Given a settlement with life insurance,
> when `SEND_INSURANCE_CONFIRM` runs,
> then LSS must call the big mum payment API with the order number from the application record.

**AC2 — Routes to Belly for non-life insurance**

> Given a settlement with non-life insurance (VMI or CMI),
> when `SEND_INSURANCE_CONFIRM` runs,
> then LSS must call the Belly payment API with the order number from the application record.

**AC3 — Both systems called when settlement has both insurance types**

> Given a settlement with both life and non-life insurance,
> when `SEND_INSURANCE_CONFIRM` runs,
> then LSS must call big mum and Belly independently.

**AC4 — Confirmation reference is stored**

> Given an insurance API call that returns success,
> then LSS must store the confirmation reference against the settlement record.

**AC5 — One insurance call failure does not block the other**

> Given both big mum and Belly are called and one fails,
> then the successful call is marked complete and the failed call is logged and marked failed independently.

---

## Step Detail: UPDATE_FACILITY

> **Fire-and-forget** — runs after `LoanSettled` is published. Condition: `HAS_FACILITY`.

### Purpose

Onigiri creates the facility record in Ectoplasm before calling LSS. LSS links the new loan account to that facility by patching it with the new loan account ID.

No facility is closed. `previous_facility_id` is not set.

### Request

LSS calls `PATCH` on the facility in Ectoplasm:

| Field | Value |
|-------|-------|
| `loan_id` | Loan account ID returned by Gringotts from `CREATE_LOAN` |

The facility ID is obtained from the application record (via `application_uuid`).

### Success and Failure Handling

| Outcome | Action |
|---------|--------|
| Ectoplasm returns success | Step marked complete. |
| Ectoplasm returns error | Failure logged. Step marked failed. Investigated and fixed separately. |

> Retry policy TBD.

### Acceptance Criteria

**AC1 — Facility is updated with the new loan account ID**

> Given a settlement with `HAS_FACILITY`,
> when `UPDATE_FACILITY` runs,
> then LSS must PATCH the facility in Ectoplasm with the `loan_id` of the new Gringotts loan account.

**AC2 — No facility is closed**

> Given a settlement with `HAS_FACILITY`,
> when `UPDATE_FACILITY` runs,
> then no existing facility must be closed and `previous_facility_id` must not be set.

**AC3 — Step skipped when no facility**

> Given a settlement without `HAS_FACILITY`,
> then `UPDATE_FACILITY` must not run and no call is made to Ectoplasm.
