# QA Technical Assessment — Retail Banking App

Enterprise-grade Quality Engineering validation portfolio for a Retail Banking Application, focusing on system resiliency, data integrity, and backend architectural test coverage for core financial transaction cycles.

---

## 📁 Repository Structure

```text
.
├── README.md
├── ai-usage.md
├── task-1/
│   └── test-strategy-and-coverage-design.md
├── task-2/
│   ├── exploratory-testing-and-bug-reports.md
│   └── evidence/
│       ├── bug_01_transfer_404.png
│       └── bug_02_loan_500.png
├── task-3/
│   ├── api-analysis-and-governance.md
│   ├── currency-data.json
│   ├── parabank-fx-rates-suite.postman_collection.json
│   └── rates-environment.json
└── task-4/
    └── framework-architecture-design.md
```

---
## ⏱️ Technical Delivery Timeline

* **Task 1 — Test Strategy & Coverage Design:** 90 minutes
  * *Scope:* Requirement analysis, question metrics, risk prioritization matrices, 12-test-case boundary design, coverage boundaries, and release quality gates.
* **Task 2 — Exploratory Testing & Bug Reporting:** 20 minutes
  * *Scope:* Time-boxed session execution on live ParaBank instances, structural root-cause isolation, business impact telemetry log, and future roadmap scoping.
* **Task 3 — API Testing & Integration Governance:** 70 minutes *(Note: Includes an extra 20 minutes of operational client profiling to map dataset query-runner behaviors across updated Postman interface layers).*
  * *Scope:* Data-Driven Postman collection engine construction, schema contract assertions, request chaining loops, and upstream dependency security analysis.
* **Task 4 — Automation Design & Test Architecture:** 50 minutes *(Note: Includes an extra 15 minutes of collaborative architectural spikes to design the abstract Semantic Parent-to-Child Anchor strategy, sight unseen due to missing visuals).*
  * *Scope:* Composite Page Component designs, Semantic Parent-to-Child Anchor strategies, dynamic DOM rehydration gatekeepers, self-healing data models, and tooling trade-off evaluation.



---

## ❓ Analytical Foundations & Strategic Gaps

### Task 1 — Local Fund Transfer Requirement Analysis
* **Transfer Limits Boundaries:** Clarified ambiguous behavior around limits, ensuring the per-transaction limit is inclusive and defining exactly how the daily limit is accumulated across multiple transactions.
* **Fee Calculation & Charging Rules:** Defined assumptions for fee coverage by available balance and validated the specific ledger behavior on failed transfers.
* **OTP Lifecycle & Security Bounds:** Formulated explicit assumptions for OTP validity windows, transaction binding payload signatures, retry lockouts, expiration limits, and anti-replay token reuse.
* **Duplicate Submissions Handling:** Defined expected behavior for duplicate submissions and structured explicit API Gateway idempotency checks.
* **Resiliency & Timeout States:** Defined handling of a Core Banking System (CBS) debit followed by a network timeout or unknown downstream outcome.
* **Transaction State Machine Transitions:** Mapped assumptions for transaction states, automated auto-reversals, and end-of-day reconciliation queues.
* **Compliance & Sanctions Controls:** Defined assumptions around beneficiary eligibility, AML/fraud blacklists, cutoff/calendar day behavior, and notification failure tolerances.
* **Critical Requirement Loop Gap (AC6):** Identified a massive gap in `AC6` where the requirement does not explicitly define the expected technical behavior or automated fund recovery path when the account is debited but the transfer subsequently fails or times out downstream.

---

## 🚫 Scope Reductions & Coverage Decisions

### Task 1 — Local Fund Transfer
* **Exhaustive UI/Device Matrices:** Browser/OS combinations were deliberately skipped due to the strict 12-test-case technical assessment constraint.
* **Full Input-Validation Permutations:** Individual data validation checks for amount text boxes, long purpose-note string inputs, special characters, and formatting fields were abstracted.
* **Exhaustive External Network Paths:** Every possible dynamic payment-rail variation, beneficiary bank switch, cut-off hour window, weekend, and holiday combination was not modeled as a separate test case.
* **Detailed Compliance Rule Books:** AML, sanctions, and fraud engine decision rules were not expanded because external system rules and data payloads were not provided.
* **Notification Provider Drops:** Full notification-provider failure/retry loops were excluded beyond enforcing the core requirement that a notification drop must never alter a successful financial ledger transaction.
* **Core Optimization Focus:** Left intentionally to be handled by parameterized API automation, manual exploratory sessions, integration staging, or future regression expansion. Testing was deliberately concentrated on **financial integrity, payload security, distributed transaction states, automated recovery, and high-risk business controls.**

---

## 🐛 Verified System Anomaly Logs

### Task 2 — Exploratory Session Findings
The exploratory testing session conducted on the ParaBank target environment exposed critical validation gaps across core payment interfaces, raising severe database integrity and customer friction threats:

* **[BUG-01] Transfer Funds Gateway Routing Breakdown (HTTP 404):** Submitting a blank payload to the amount text field breaks the backend routing framework, triggering a raw `404 Not Found` response code and leaking a generic system crash text (*"An internal error has occurred..."*) to active user sessions.
* **[BUG-02] Underwriting Microservice Uncaught Exception (HTTP 500):** Passing a zero-value parameter matrix (`$0 amount` and `$0 down payment`) inside the loan request module bypasses presentation validations and triggers a raw `500 Internal Server Error`, exposing a severe lack of defensive math exception handlers within the credit processing code layer.

---

## 🧪 Automated API Testing & Integration Governance

### Task 3 — Data-Driven Contract Framework
The API portfolio delivers a fully parameterized, data-driven automation solution validating integration boundaries with public currency quotation gateways, backed by a comprehensive security governance review:

* **Data-Driven Automation Matrix:** Constructed a single parameterized Postman blueprint that runs an automated execution loop across 4 base currency entities (`USD`, `EUR`, `GBP`, `AED`), fed directly from an external data pool ledger (`currency-data.json`), completely eliminating hardcoded request duplication.
* **Dynamic Pipeline Chaining:** Automatically harvests volatile server metadata state elements (`time_next_update_unix`) from active payloads at runtime, caching parameters inside environment tables (`rates-environment.json`) to enforce cross-request session synchronization.
* **REST Architecture Critique:** Documents a major validation critique of the gateway's error encapsulation methods—exposing the critical system risk of wrapping internal execution failures inside successful `200 OK` HTTP envelopes when parsing malformed query keys (`INVALID`).
* **Production Defenses Scoping:** Details explicit architectural specifications for moving public integrations to production banking baselines, mandating client-side caching gates, IPsec VPN/mTLS tunnel paths, and automatic circuit-breaking routines mapped to active local cached backup exchange blocks.
---
## 🧪 Automation Framework Design & Test Architecture

### Task 4 — Scalable Operations Component Framework

The automation design portfolio delivers a decoupled, high-scale framework blueprint targeting the dynamic Bank Operations Console. It focuses on robust locator maintainability, zero static sleeps, and shared-environment data isolation:

* **Composite Page Component Architecture:** Completely decouples the console workspace into localized, isolated tab modules to enforce the Single Responsibility Principle, ensuring all classes remain strictly under 100 lines.
* **Semantic Parent-to-Child Anchor Strategy:** Bypasses session-dynamic identifiers (`slot="field-145"`) by anchoring onto the unique outer wrapper via stable text labels, utilizing scoped ARIA roles locally to polymorphic textboxes, dropdowns, and checkboxes.
* **Dynamic Web Assertions & Rehydration Gates:** Eliminates brittle static sleeps via auto-polling web-first assertions and implements custom tab synchronization gatekeepers to perfectly insulate execution threads against dynamic DOM rehydration and stale element nodes.
* **Self-Healing State Isolation Strategy:** Mitigates data-overwrite risks in shared testing environments by utilizing an automated baseline-seeding API client mechanism that records existing records pre-execution and enforces a mandatory reversion cleanup block during teardown.
* **Multi-Layered Execution Architecture:** Establishes completely independent structural directory layers, segregating code into standalone browser context auth configurations, pure UI functional specs, async API contract validation specs, and Webpack-bundled TypeScript performance load scripts.
