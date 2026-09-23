# QA Technical Assessment

## Repository Structure

```text
.
├── README.md
├── ai-usage.md
├── task-1/
│   ├── README.md
│   └── test-design.md
├── task-2/
├── task-3/
├── task-4/
└── task-5/ 

---

## Time Spent

### Task 1 — Test Design & Risk Coverage

- **Time spent:** [90m]
- Time included requirement analysis, clarification of assumptions, risk prioritization, test-case design, coverage decisions, and release-readiness criteria.

---

## Questions / Assumptions

### Task 1 — Local Fund Transfer

- Clarified ambiguous behavior around **transfer limits**, including whether the per-transaction limit is inclusive and how the daily limit is accumulated.
- Defined assumptions for **fee calculation and charging**, including fee coverage by available balance and behavior on failed transfers.
- Defined ****OTP** validity, transaction binding, retry, expiration, and reuse** assumptions.
- Defined expected behavior for **duplicate submissions and idempotency**.
- Defined handling of ****CBS** debit followed by timeout or unknown downstream outcome**.
- Defined assumptions for **transaction states, reversal, and recovery**.
- Defined assumptions around **beneficiary eligibility, compliance/fraud controls, cutoff/calendar behavior, and notification failure**.
- Identified the critical requirement gap in **AC6**: the requirement does not explicitly define the expected behavior when the account is debited but the transfer subsequently fails or becomes unknown.

---

## What Was Left Out

### Task 1 — Local Fund Transfer

- Exhaustive UI/device/browser combinations were not covered due to the 12-test-case constraint.
- Full input-validation permutations for amount, purpose-note length, characters, and formatting were not individually covered.
- Every possible payment-rail, bank, cutoff, weekend, and holiday combination was not modeled as a separate test case.
- Detailed **AML**/sanctions/fraud rule permutations were not expanded because the actual decision rules and external responses were not specified.
- Full notification-provider failure/retry permutations were not expanded beyond the critical requirement that notification failure must not alter a successful financial transaction.
- Lower-risk variations were intentionally left for **parameterized automation, exploratory testing, integration testing, or future regression expansion**.
- Testing was deliberately concentrated on **financial integrity, security, transaction state, recovery, and high-risk business controls**.

---