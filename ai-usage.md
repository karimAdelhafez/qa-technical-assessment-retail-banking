 # AI Usage Documentation — Task 1

## 🧠 Human-AI Collaborative Workflow (Strategic Technical Sparring)

* The test design was developed through an iterative, bi-directional technical dialogue between the **QA Lead (Domain Expert)** and the **AI (Systems Architecture Co-Pilot)**. The process was not limited to generating documentation; both sides were used to challenge, refine, and stress-test the proposed coverage.

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

## 🎛️ 1. Prompts Used
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

## ✅ 2. What I Kept & Rationale
Because the initial prompting sequence established highly restrictive technical boundaries, the collaborative output achieved excellent engineering alignment on the core logic. I retained:

* **Distributed Fault Resiliency Steps:** Retained the edge-case handling sequences validating how the system transitions when the network drops exactly between a successful CBS account debit and a clearing switch ACK.
  * *Rationale:* Essential for confirming distributed ledger integrity and tracking `Pending/Unknown` states.
* **Service-Layer Idempotency Controls:** Retained the verification checks targeting Idempotency-Key validation on the API Gateway.
  * *Rationale:* Bullets an enterprise-grade solution to eliminate double-debit exposure from rapid multi-clicks.
* **Boundary Cap Expressions:** Retained the rapid calculation models mapping out exact positive/negative limits (`10,000.01` and `20,000.01`) directly tracing back to `AC4`.

---

## ❌ 3. What I Rejected & Refined (Human-Led Quality Control)

* **No Material Rejections:** Because my engineering prompts enforced strict constraints upfront (explicitly ordering the tool to bypass conversational boilerplate, eliminate standard front-end UI fluff like button colors, and stick purely to dense backend matrices), **there were no irrelevant or low-quality outputs generated to be rejected.**
* **Structural Enforcement:** The only adjustments made during the workflow were structural filters—reminding the tool to keep its text tightly wrapped and formatted in clean Markdown to maximize scannability for secondary automation engineers.
* **Refined: Vague Test Case Titles & Unstructured Steps:** While the core technical logic was sound from the start, the AI's first formatting pass fell short of lead-level documentation standards:
  * *Vague Titles:* The AI initially spit out simple headers like `TC-02 — Limit Check`. I rejected these and ordered a rewrite to use **Comprehensive Titles** detailing the Action, Component Boundary, and Expected Operational Outcome.
  * *Sloppy Steps:* The tool initially combined front-end button triggers with database logs in a loose sequence. I stepped in to enforce **QA Engineering Best Practices**, forcing clean isolation of preconditions, distinct sequential action steps, and an explicit separation of **UI/Frontend vs. Backend/CBS Ledger** expected results.

---

## 🔍 4. Technical Interview Walk-Through Notes
* **Decisions Made:** I chose to treat the AI as a technical compilation engine rather than blindly accepting its structure. Driving the tool with deep backend parameters from the first sentence prevented it from drifting into presentation-layer scripts, while my structural adjustments ensured the matrix is fully maintainable and automation-ready.
* **Alternatives Considered:** Allowing the AI to freely generate a standard generic test plan first, then filtering it. I rejected this because it wastes context tokens and degrades the engineering precision required for a Lead-level submission.

