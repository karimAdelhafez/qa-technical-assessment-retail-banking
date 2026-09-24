 # AI Usage Documentation

This document logs the collaborative technical sparring workflow between myself (QA Lead) and the AI tool acting as Quality Engineering & System Resiliency Co-Pilot, organized clearly by task.

---

## 🧠 Human-AI Collaborative Workflow (Strategic Technical Sparring)

* The test design was developed through an iterative, bi-directional technical dialogue between the **QA Lead** and the **AI (Quality Engineering & System Resiliency Co-Pilot)**. The process was not limited to generating documentation; both sides were used to challenge, refine, and stress-test the proposed coverage.

* I established the core **Retail Banking baseline**, including:
  * Core Banking System (CBS) ledger and debit rules.
  * Available-balance and account-concurrency considerations.
  * AML, sanctions, and fraud screening checkpoints.
  * Central Payment/Clearing Switch responsibilities.
  * Instant versus ACH clearing flows.
  * Transaction lifecycle states, including successful settlement, pending/unknown outcomes, reversal, and reconciliation.

* The AI actively challenged my initial test ideas by cross-referencing them against common **distributed-system failure modes** and asynchronous processing risks.

* One important example was the expansion of the timeout scenario. Rather than treating a timeout as a simple API failure, the discussion analyzed the more dangerous sequence where:
  * The CBS successfully debits the customer's account.
  * The network connection drops or times out.
  * The downstream clearing/payment component has not yet returned its acknowledgement.
  * The original transaction outcome therefore becomes temporarily unknown.

* This led to explicit consideration of **Pending/Unknown states, idempotent retries, reconciliation, and controlled reversal/recovery**, rather than simply displaying a generic transfer error.

* I audited the resulting suggestions against the assessment's strict **12-test-case boundary**, retaining scenarios that materially affected financial integrity, security, resiliency, or transaction correctness.

* Generic or lower-risk UI scenarios were deliberately culled when they competed with higher-value distributed-system and backend scenarios.

---

### 📊 The Reverse AI Workflow Mapping
To maximize execution speed without sacrificing domain ownership, the collaborative process followed this strict technical pipeline:

```text
    [My Banking / QA Domain Knowledge] 
    (CBS Ledger Posting, AML Holds, Payment Switch Rails, OTP Binding)
                   │
                   ▼
       [My Architectural Model]
    (Asynchronous Timeout Sequences & Concurrency Windows)
                   │
                   ▼
     [Explicit Technical Directives]
    (Enforce Idempotency-Keys, Boundary Limits, Tabular Formats)
                   │
                   ▼
       [AI Systems Co-Pilot Sparring]
    (Distributed Failure Intersections & Formatting Engine)
                   │
                   ▼
       [Final Lead Review & Sign-Off]
```

---
## 📊 Task 1 — Test Strategy & Coverage Design

### 🎛️ 1. Prompts Used
The workflow was driven by executing technically precise, highly scoped domain prompts to prevent generic AI drift and enforce architectural constraints from the start:

* **The Master Execution Prompt:**
  ```text
  Act as an elite Principal Quality Engineering Architect and Banking Domain Expert. I am a Senior SDET / QA Lead working on a time-boxed technical assessment for a Retail Banking App (Local Fund Transfer Feature). We have already aligned on the core architecture (CBS, ACH vs Instant Rails, ISO 20022, Idempotency, and Fraud/AML Integration). You must assist me in executing this assessment at a world-class Lead Level, ensuring maximum technical depth with zero fluff to strictly conserve tokens.
  [STRICT EXECUTION PROTOCOL]:
  1. Pure English Output: Deliver all technical terms, frameworks, and architecture verifications using enterprise-grade English..
  2. Step-by-Step Flow: Do NOT dump all answers at once. Answer exactly ONE question or sub-task at a time, then wait for my signal to proceed to the next.
  3. Format: Use compact markdown tables and short, punchy, single-fragment bullet points. Avoid introductory or concluding pleasantries.
  4. Postcondition Rule: Once we agree on the final solution of any task, you must explicitly summarize the outputs into clear file blocks and wait for my signal.
  ```

---

### ✅ 2. What I Kept & Rationale
Because the initial prompting sequence established highly restrictive technical boundaries, the collaborative output achieved excellent engineering alignment on the core logic. I retained:

* **Distributed Fault Resiliency Steps:** Retained the edge-case handling sequences validating how the system transitions when the network drops exactly between a successful CBS account debit and a clearing switch ACK.
  * *Rationale:* Essential for confirming distributed ledger integrity and tracking `Pending/Unknown` states.
* **Service-Layer Idempotency Controls:** Retained the verification checks targeting Idempotency-Key validation on the API Gateway.
  * *Rationale:* Bullets an enterprise-grade solution to eliminate double-debit exposure from rapid multi-clicks.
* **Boundary Cap Expressions:** Retained the rapid calculation models mapping out exact positive/negative limits (`10,000.01` and `20,000.01`) directly tracing back to `AC4`.

---

### ❌ 3. What I Rejected & Refined (Human-Led Quality Control)

* **No Material Rejections:** Because my engineering prompts enforced strict constraints upfront (explicitly ordering the tool to bypass conversational boilerplate, eliminate standard front-end UI fluff like button colors, and stick purely to dense backend matrices), **there were no irrelevant or low-quality outputs generated to be rejected.**
* **Structural Enforcement:** The only adjustments made during the workflow were structural filters—reminding the tool to keep its text tightly wrapped and formatted in clean Markdown to maximize scannability for secondary automation engineers.
* **Refined: Vague Test Case Titles & Unstructured Steps:** While the core technical logic was sound from the start, the AI's first formatting pass fell short of lead-level documentation standards:
  * *Vague Titles:* The AI initially spit out simple headers like `TC-02 — Limit Check`. I rejected these and ordered a rewrite to use **Comprehensive Titles** detailing the Action, Component Boundary, and Expected Operational Outcome.
  * *Sloppy Steps:* The tool initially combined front-end button triggers with database logs in a loose sequence. I stepped in to enforce **QA Engineering Best Practices**, forcing clean isolation of preconditions, distinct sequential action steps, and an explicit separation of **UI/Frontend vs. Backend/CBS Ledger** expected results.

---

## 🐛 Task 2 — Exploratory Testing & Bug Reporting

### 🎛️ 1. Prompt Used
```text
Act as an elite Principal QA Architect and Banking Domain System Reviewer. I am a QA Lead executing a time-boxed "Task 2 — Exploratory Testing & Bug Reporting" assignment. I have already performed the session and captured concrete functional defects. 

You must act as my technical sparring partner to brainstorm, polish, and structure these findings into world-class, high-density engineering bug reports with zero boilerplate filler.

[STRICT PROTOCOL FOR TOKEN CONSERVATION & QUALITY]:
1. **Interactive Review:** Do NOT generate any test plans or empty tables yet. Review the raw bugs I am about to paste one by one.
2. **Lead-Level Enhancement:** For each defect I share, you must immediately analyze it and provide:
   * A concise, high-context title (Action + Component + Root Cause).
   * The underlying technical/architectural root cause (e.g., State Machine imbalance, DB lock failure, improper validation layer).
   * The downstream systemic impact across three boundaries: Customer, Bank Ledger, and Compliance/Audit.
3. **Format:** Use short, punchy single-sentence fragments and compact layouts. No greetings or pleasantries.
```

### ✅ 2. What I Kept & Rationale
* **API Error Classifications:** I kept the deep-dive architectural isolation of the `404 Not Found` routing breakdown and the `500 Internal Server Error` code definitions.
  * *Why:* It cleanly demonstrates to the reviewer how unhandled backend exceptions threaten database connection pools, thread safety, and core ledger integrity under edge inputs.
* **Risk-Based Exploration Tour:** I kept the strategic framing of the execution approach around a "Transactional Lifecycle Tour" targeting balance-altering endpoints.
  * *Why:* It proves that the testing effort was structured around architectural risk management rather than random UI clicking.

### ❌ 3. What I Rejected & Refined (My Corrections)
* **Caught Mismatched Environment Parameters:** The AI accidentally injected a generic corporate homepage link (`parasoft.com`) inside the environment parameters block of both bugs. I caught this error during my active review and manually overrode the text to map the exact, live staging application paths (`://parasoft.com...`) to prevent environment invalidation.
* **Overrode Generic Technical Explanations:** The tool initially generated text-heavy descriptions regarding general standard HTTP validations. I stepped in and merged my explicit reasoning—identifying the root issue as a failure to handle input validations on both the frontend and backend layer—to ensure the reports read in a genuine, human engineering voice.
* **Separated Business Justifications:** The AI initially combined the priority and severity risk analysis into single paragraphs. I rejected that format and forced them into explicit, distinct single-line text statements to achieve 100% compliance with the assessment grading checklist.
---

## 💸 Task 3 — API Testing

### 🧠 Human Guidance & Testing Scope
While executing this task, I explicitly directed the technical sparring sessions to ensure the automated suite and written analysis covered my core QA Lead criteria: **Happy Paths, Edge Cases, Critical Banking Scenarios, Input Validation Matrices (Data Types & Boundaries), Missing/Mandatory Parameters, and Authentication Layer Constraints**. This human-led scope forced the output to focus entirely on deep banking risks rather than generic CRUD API responses.

### 🎛️ 1. Prompts Used
* **The Master API Strategy & Collection Prompt:**
  ```text
  Act as a Principal QA Architect. Help me execute a time-boxed technical assessment for a Retail Banking App transfer feature. We are testing a stack that involves a CBS ledger, instant payment rails, and AML screening. Give me zero conversational filler. Output must be in dense markdown tables and short bullet points. Let's work step-by-step.
  ```
* **The Payment Endpoint Analysis Prompt:**
  ```text
  Act as a Senior Banking QA Reviewer. I am analyzing a balance-altering POST payments endpoint for out-of-bank transfers. Help me structure my risk strategy, idempotency-key handling logic, and end-to-end ledger verification mechanisms into high-density engineering responses. Focus on structural boundaries (floating-point precision, currency cross-contamination, pessimistic DB locking) and architectural constraints. Keep it punchy and concise.
  ```

### ✅ 2. What I Kept & Rationale
* **JSON Schema Contract Assertions (Part A):** Retained the exact 12-check validation framework to verify structural data types, numeric floats, and parameter consistency inside Postman.
  * *Why:* It cleanly demonstrates dynamic schema verification over simple status code validation.
* **Idempotency and Resilience Logic (Part B):** Retained the technical decoupling of error states, differentiating retry behaviors for network timeouts versus unhandled `503 Service Unavailable` server responses.
  * *Why:* It highlights specialized domain knowledge regarding API gateway caching layers and double-debit prevention in banking architectures.

### ❌ 3. What I Rejected & Refined (My Corrections)
* **Zero Technical Drift via Strict Initial Prompting:** Because my initial master prompts enforced aggressive, zero-filler engineering criteria from the absolute start, the AI co-pilot executed all transaction validation frameworks correctly on the first pass, leaving zero low-quality or out-of-scope text blocks to be rejected.
* **Refined Dynamic Collection Execution:** The only adjustments made were during the sandbox runtime configuration, where I manually guided the script execution paths to implement explicit `postman.setNextRequest()` routing loops to ensure all sequential chaining layers ran flawlessly across all iterations, achieving a clean **48/48 automated test pass rate**.
---

## 📈 Task 4 — Automation Design (No Code)

### 🧠 Human Guidance & Collaborative Brainstorming
* **The Dynamic Wrapper Obstacle:** When I first looked at the Bank Operations Console requirements, I noticed a huge technical blocker: all the input fields use random session IDs like `slot="field-145"`, and the form components are repeated across multiple tabs. Writing traditional locators here would make the tests fail constantly. To be completely honest, I wasn't sure how to cleanly bypass this dynamic wrapper issue at first.
* **The Brainstorming Breakthrough:** I used the AI co-pilot to run a technical design spike and brainstorm options. Together, we came up with a **Semantic Parent-to-Child Anchor Strategy**. Instead of chasing the broken dynamic IDs, we anchor onto the stable text labels (like "Transfer Limits") to isolate that specific field container first. I then forced the logic to use scoped Playwright ARIA roles inside that bounded box to select textboxes, dropdowns, or checkboxes without any ambiguity.

### 🎛️ 1. Prompts Used
* **The Master Automation Design Prompt:**
  ```text
  Act as a Senior Automation Architect specializing in Playwright and modern QA frameworks. I need you to design a scalable automation solution for Task 4 — Automation Design. We are automating regression coverage for a Bank Operations Console with multiple tabs. Each tab contains text, checkbox, and dropdown fields inside stable parent wrappers with dynamic child identifiers. Do not use dynamic IDs. Detail the project structure, locator strategy, dynamic DOM handling, test data strategy, and tool stack trade-offs. Present as structured sections with short snippets.
  ```

### ✅ 2. What I Kept & Rationale
* **Composite Page Component Pattern:** I kept the strategy of breaking down each tab into its own sub-component file inside the components directory.
  * *Why:* It stops the codebase from turning into a giant, unreadable 900-line class file and keeps files under 100 lines for easy maintenance.
* **API Baseline Capture & Teardown Lifecycles:** I kept the dynamic environment data strategy where we query configurations via API first, modify via UI, and revert everything back to normal.
  * *Why:* This is the only realistic way to run parallel tests on a shared test environment without messing up data for other teams.

### ❌ 3. What I Rejected & Refined (My Corrections)
* **Separating Tests by Protocol Layers:** The co-pilot initially dumped all the tests into a single folder. I stepped in and organized the structure to cleanly separate tests into isolated directories: **Auth setups, UI tests, API contract specs, and k6 concurrency scripts**.
* **Adding Real-World Tooling Trade-offs:** The initial tool analysis was too generic. I refined the tooling stack matrix to focus strictly on **Playwright (TypeScript)** and added explicit, practical constraints detailing the **Webpack bundling steps** needed to make TypeScript run inside **k6's Go-based engine**.

---

## 📈 Task 5 — Quality Strategy & Release Judgement

### 🧠 Human Guidance & Collaborative Brainstorming
* **The Release Pressure Confrontation:** Facing a strict 3-working-day marketing release constraint under a broken 68% regression rate and unvetted FX integrations represents a classic corporate delivery trap. I actively blocked the co-pilot's initial attempts to generate standard, text-heavy testing checklists or propose conditional "go-live" frameworks. 
* **The Resource & Risk Breakthrough:** I initiated a technical sparring session to inject my real-world management experience into the context window. I directed the tool to build an active, tenure-based resource allocation matrix—strategically separating new joiners to handle isolated tasks (UAT client embedding and flaky test triage) while locking my core veteran engineering assets onto deep backend ledger validations and shared customer profile bugs.

### 🎛️ 1. Prompts Used
* **The Master Quality Strategy Memo Prompt:**
  ```text
  Act as a Senior Automation Architect and QA Director with 30 years of enterprise risk experience. I need you to design a scalable strategy and release judgment memo for Task 5. Scenario involves a strict 3-day go-live window with a 68% regression pass rate, 2 critical shared KYC profile service bugs, unvetted FX staging code, and an un-signed client UAT. Provide a strict Go/No-Go decision, a 72-hour priority team allocation plan splitting tasks by engineer project tenure, upstream/external communication matrices, and long-term automated CI/CD quality gates. Present in a clean human technical style with zero fluff.
  ```

### ✅ 2. What I Kept & Rationale
* **Absolute NO-GO Release Block:** Retained the strict, un-compromised release halt based on financial ledger vulnerabilities and unsigned client environments.
  * *Why:* In transactional banking systems, regulatory compliance and data integrity completely override commercial marketing windows.
* **Automated CI/CD Quality Gates & Anti-Metrics:** Retained the long-term operational framework changes mandating a 95% master branch merge blocker paired with a complete ban on individual developer bug counts.
  * *Why:* It protects corporate engineering velocity and upholds Goodhart's Law, shifting focus from artificial volumetric numbers to real systemic quality.

### ❌ 3. What I Rejected & Refined (My Corrections)
* **Overrode Volumetric Engineering Metrics:** The tool initially attempted to suggest tracking code-coverage lines and bug density counts per individual code owner. I completely rejected this conversational baseline, reframing the anti-metrics section to explicitly cite **Goodhart's Law** and replacing the fluff with corporate governance metrics like **UAT Burn-down Velocity and Defect Leakage Rates**.
* **Enforced Practical Engineering Realities:** I stepped in during the action plan phase and manually mapped out the structural distribution of my 4-engineer team based strictly on project lifecycle tenure, preventing the AI from creating generic multi-browser UI testing tasks during a high-stakes core banking release crisis.
