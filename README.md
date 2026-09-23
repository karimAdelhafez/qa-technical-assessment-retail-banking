# QA Technical Assessment — Retail Banking App

Enterprise-grade Quality Engineering validation portfolio for a Retail Banking Mobile Application, focusing on system resiliency, data integrity, and backend architectural test coverage for the Local Fund Transfer feature.

## 📁 Repository Structure


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
```

---

## ⏱️ Time Spent

### Task 1 — Test Design & Risk Coverage
* **Time spent:** 90 minutes
* **Scope included:** Requirement analysis, clarification of assumptions, risk prioritization, test-case design, coverage decisions, and release-readiness criteria.

---

## ❓ Questions / Assumptions

### Task 1 — Local Fund Transfer
* **Transfer Limits boundaries:** Clarified ambiguous behavior around limits, ensuring the per-transaction limit is inclusive and defining exactly how the daily limit is accumulated across multiple transactions.
* **Fee Calculation & Charging rules:** Defined assumptions for fee coverage by available balance and validated the specific ledger behavior on failed transfers.
* **OTP Lifecycle & Security bounds:** Formulated explicit assumptions for OTP validity windows, transaction binding payload signatures, retry lockouts, expiration limits, and anti-replay token reuse.
* **Duplicate Submissions handling:** Defined expected behavior for duplicate submissions and structured explicit API Gateway idempotency checks.
* **Resiliency & Timeout states:** Defined handling of a Core Banking System (CBS) debit followed by a network timeout or unknown downstream outcome.
* **Transaction State Machine transitions:** Mapped assumptions for transaction states, automated auto-reversals, and end-of-day reconciliation queues.
* **Compliance & Sanctions controls:** Defined assumptions around beneficiary eligibility, AML/fraud blacklists, cutoff/calendar day behavior, and notification failure tolerances.
* **Critical Requirement Loop Gap (AC6):** Identified a massive gap in `AC6` where the requirement does not explicitly define the expected technical behavior or automated fund recovery path when the account is debited but the transfer subsequently fails or times out downstream.

---

## 🚫 What Was Left Out (Out of Scope)

### Task 1 — Local Fund Transfer
* **Exhaustive UI/Device matrices:** Browser/OS combinations were not covered due to the strict 12-test-case technical assessment constraint.
* **Full Input-Validation permutations:** Individual data validation checks for amount text boxes, long purpose-note string inputs, special characters, and formatting fields were abstracted.
* **Exhaustive External Network paths:** Every possible dynamic payment-rail variation, beneficiary bank switch, cut-off hour window, weekend, and holiday combination was not modeled as a separate test case.
* **Detailed Compliance rule books:** AML, sanctions, and fraud engine decision rules were not expanded because external system rules and data payloads were not provided.
* **Notification Provider drops:** Full notification-provider failure/retry loops were excluded beyond enforcing the core requirement that a notification drop must never alter a successful financial ledger transaction.
* **Lower-Risk variations:** Left intentionally to be handled by parameterized API automation, manual exploratory sessions, integration staging, or future regression expansion.
* **Core Focus:** Testing was deliberately concentrated on **financial integrity, payload security, distributed transaction states, automated recovery, and high-risk business controls.**
