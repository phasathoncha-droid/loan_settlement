# Loan Settlement System — Requirements Index

**System**: Loan Settlement System (LSS)
**Status**: Draft
**Last Updated**: 2026-05-28

---

## Documents

| Document | Description |
|----------|-------------|
| [LSS_Overview.md](LSS_Overview.md) | System overview — what LSS does, systems, business rules, orchestration model, events |
| [LOAN_NEWBOOK.md](LOAN_NEWBOOK.md) | Full spec for New Book (`LOAN_NEWBOOK`) |
| [LOAN_TOPUP.md](LOAN_TOPUP.md) | Full spec for Top-Up (`LOAN_TOPUP`) |
| [LOAN_RESTRUCTURE.md](LOAN_RESTRUCTURE.md) | Full spec for Restructure (`LOAN_RESTRUCTURE`) |

---

## How to Read

**New to LSS?** Start with [LSS_Overview.md](LSS_Overview.md). It explains settlement concepts, business rules (Rule 1/2/3), the orchestration model, and events. Read this first.

**Working on a specific loan product?** Go directly to the product file. Each file is self-contained — it includes the flow diagram, phase/step configuration, and full step details with acceptance criteria.

**Adding a new loan product?** See [Adding a New Loan Product](#adding-a-new-loan-product) below.

---

## Loan Product Summary

| Product | Event Code | Disbursement | Gringotts API | SAP New Contract | SAP Close Contract | Insurance |
|---------|-----------|-------------|---------------|------------------|--------------------|-----------|
| New Book | `LOAN_NEWBOOK` | NO_FTR / HAS_FTR | `CREATE_LOAN` | A100 + conditional | — | ✅ Yes |
| Top-Up | `LOAN_TOPUP` | NO_FTR / HAS_FTR | `TOPUP_LOAN` | A103 + A100 + conditional | ✅ Yes | ✅ Yes |
| Restructure | `LOAN_RESTRUCTURE` | NO_FTR only | `RESTRUCTURE_LOAN` | A103 + A997 + conditional | ✅ Yes | ❌ No |

---

## Adding a New Loan Product

When a new loan product type is added to LSS:

1. **Create a new file** named `LOAN_<EVENTCODE>.md` in this folder (e.g. `LOAN_EARLYCLOSE.md`)
2. **Follow the same structure** as the existing product files:
   - What It Is (settlement balance equation)
   - Flow diagram(s)
   - Phase/Step Configuration table
   - Success Definition table
   - One section per step with: Purpose, key fields/rules, Success and Failure Handling, Acceptance Criteria
3. **Add a row** to the Loan Product Summary table above
4. **Update CLAUDE.md** — add the new product to the loan product list so future sessions are aware of it
