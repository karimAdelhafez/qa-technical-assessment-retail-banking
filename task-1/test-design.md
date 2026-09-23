# Task 1 — Test Design & Risk Coverage

## 📋 1. Task 1.1 — Questions & Assumptions Matrix

| #       | Question to BA / PO / Business                                                                                                  | Why it matters                                                                   | Assumption if unanswered                                                                                                         |
| ------- | ------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Q1**  | Is the transfer **same-bank only**, or does it include **other-bank local transfers**? Which payment rail is expected?          | Determines routing, processing time, clearing, settlement, and failure behavior. | Assume local interbank transfers are supported and the backend selects the appropriate rail.                                     |
| **Q2**  | Does the **20,000 daily limit** count only successful transfers, or also pending/failed/rejected transfers? When does it reset? | Directly affects limit enforcement and boundary scenarios.                       | Count successfully executed transfers; reset at the bank-defined business-day boundary.                                          |
| **Q3**  | Does the **10,000 per-transaction limit** apply to the transfer principal only, or principal + fee?                             | Critical for boundary cases such as 9,990 + fee.                                 | Assume the limit applies to the transfer principal only.                                                                         |
| **Q4**  | Is the 20,000 daily limit **per customer** or **per source account**?                                                           | Otherwise a customer could potentially bypass the limit using multiple accounts. | Assume it is a customer-level limit across eligible source accounts.                                                             |
| **Q5**  | Is the transfer fee deducted from the same source account? Does the fee require additional available balance?                   | Determines the actual debit amount and insufficient-balance behavior.            | Fee is debited from the source account and must be covered by available balance.                                                 |
| **Q6**  | When is the fee applied: before OTP, after OTP, or at execution? Is it charged if the transfer fails?                           | Prevents incorrect fee charging for unsuccessful transactions.                   | Fee is applied only when the transfer is successfully executed.                                                                  |
| **Q7**  | What is the **OTP validity period, maximum attempts, resend policy, and transaction binding**?                                  | Defines authentication security and replay protection.                           | OTP has a short configured validity period, limited attempts, and is bound to the specific transfer.                             |
| **Q8**  | What happens after an expired/incorrect OTP or too many failed attempts?                                                        | Defines authentication failure states and abuse protection.                      | Transfer is not executed; excessive failures trigger configured rate-limit/lock behavior.                                        |
| **Q9**  | What happens if the customer submits the transfer twice or the client retries the API request?                                  | Prevents duplicate financial transactions.                                       | The same logical transfer must have **one financial effect**, enforced through idempotency.                                      |
| **Q10** | What happens if the account is debited but the mobile app receives a timeout/no response?                                       | This is a critical financial-integrity scenario.                                 | Transaction moves to **Pending/Unknown** and is resolved through backend status/reconciliation rather than blindly retried.      |
| **Q11** | Which source accounts are eligible, and is validation based on **available balance** or ledger balance?                         | Defines account eligibility and prevents invalid/overdraft transfers.            | Only active eligible accounts with sufficient available balance can transfer.                                                    |
| **Q12** | Must the beneficiary remain **active/verified** at execution time? What if the beneficiary was disabled after registration?     | A registered beneficiary is not necessarily valid at execution time.             | Beneficiary must be active and eligible when the transfer is executed.                                                           |
| **Q13** | Are **AML, sanctions, or fraud checks** performed? Can a transaction be placed on hold?                                         | These controls may introduce additional states beyond simple success/failure.    | Compliance/risk checks may block, hold, or reject a transfer before completion.                                                  |
| **Q14** | What are the authoritative transaction states: **Success, Failed, Pending, Reversed, Unknown**?                                 | Required for state-transition testing and recovery scenarios.                    | At minimum: `Pending → Success` or `Pending → Failed`; reversal applies when money was debited but the transfer cannot complete. |
| **Q15** | Is SMS notification failure allowed to make the financial transaction fail?                                                     | Notification failure should not normally create a second financial transaction.  | SMS is non-blocking; a successful transfer remains successful if notification delivery fails.                                    |
| **Q16** | What exact data must appear on the confirmation screen and SMS?                                                                 | Defines functional validation, data integrity, and sensitive-data masking.       | Both contain approved transfer details and transaction reference; sensitive account data is masked.                              |
| **Q17** | Does the optional purpose note have maximum length, character restrictions, or mandatory rules?                                 | Required for input validation, data integrity, and security testing.             | Optional field with a defined maximum length and server-side validation.                                                         |
| **Q18** | Are there **cutoff times, weekends, holidays, or clearing windows** affecting processing?                                       | Important for transfers processed through batch/clearing rails.                  | Processing follows the configured bank/payment-scheme calendar and cutoff rules.    |

### 👑 Project Prioritization Strategy (Priority if Time is Limited)

#### 🟢 P0 — Financial Integrity

- **Limits:** Validating exact daily versus per-transaction boundary semantics.
- **Fees:** Checking runtime calculation models, transaction timing, and fee recovery/failure behavior.
- **Duplicate Submissions:** Enforcing idempotent request handling to prevent duplicate financial transactions during retries, double-clicks, or repeated submissions.
- **Resiliency Breaks:** Managing a successful debit paired with an immediate downstream timeout or unknown outcome.
- **Ledger Synchronization:** Tracking transaction-state consistency and controlled recovery/reversal mechanisms across financial processing components.

#### 🔴 P1 — Security & Compliance

- **OTP Lifecycle:** Validating OTP expiration, transaction binding, retry limits, lockout/rate-limiting, and anti-replay behavior.
- **Beneficiary Eligibility:** Reviewing active, suspended, inactive, or otherwise ineligible destination accounts before execution.
- **AML / Sanctions / Fraud Controls:** Verifying required screening and risk-control decisions, including PASS, HOLD, and REJECT outcomes where applicable.

#### 🟡 P2 — Operational / UX

- **SMS Failure Behavior:** Ensuring notification delivery failures do not alter a successfully completed financial transaction.
- **Confirmation Data Mapping:** Verifying that transaction references and financial details displayed to the customer accurately reflect the authoritative transaction record.
- **Purpose-Note Validation:** Validating bounded input and appropriate server-side handling, including security-oriented input validation such as XSS protection where applicable.
- **Cutoff / Holiday Behavior:** Validating state transitions and expected processing behavior across configured clearing windows, cutoffs, weekends, and holidays.

---

### 🚨 Critical Requirement Loophole Gap (AC6 — Failure Loop)

The highest-risk ambiguity in the supplied requirement is explicitly inside **AC6 — Failure**.

### Asynchronous Error Flow

```text
       [Transfer Request]
               ↓
        [CBS Debit = SUCCESS]
               ↓
 [Payment Processing = TIMEOUT / UNKNOWN]
               ↓
        CRITICAL GAP:
   What happens to the customer's money?
```

This must be clarified because **"transfer failed" cannot safely mean "perform another transfer."** The system requires an explicit rule to transition **Pending / Unknown ➡️ Settled or Reversed** via an automated, idempotent recovery process.


## 📊 2. Task 1.2 — Risk Assessment & Impact Strategy

    I would prioritize risks based on **financial impact, security exposure, customer impact, regulatory/reputational impact, and likelihood of failure**.

| Priority | Risk Area | What Could Go Wrong | Customer Impact | Bank Impact | Delivery Partner Impact |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **P0** | **Duplicate Transfer / Idempotency** | Same request is processed twice due to double-click, retry, or network timeout | Customer loses money twice | Financial loss, complaints, reconciliation issues, regulatory exposure | Critical production defect; loss of trust in our engineering quality |
| **P0** | **Debit vs Transfer Outcome** | Account is debited but downstream transfer fails or outcome becomes unknown | Money temporarily or permanently unavailable | Financial exposure, manual investigation, compensation/reversal costs | Severe defect demonstrating inadequate distributed-transaction handling |
| **P0** | **Incorrect Balance / Concurrency** | Concurrent transfers bypass available-balance checks | Overdraft or unexpected balance reduction | Financial loss and ledger inconsistency | High-severity data-integrity defect |
| **P0** | **Limits Enforcement** | Daily or per-transaction limits are calculated incorrectly or can be bypassed | Customer may transfer more than intended/allowed | Policy/control violation and potential financial exposure | Defect in core business-control implementation |
| **P0** | **OTP / Authentication** | OTP can be reused, bypassed, guessed, or applied to another transaction | Unauthorized transfer / account compromise | Direct financial loss, fraud, security and regulatory consequences | Major security defect and potential contractual escalation |
| **P1** | **Fee Calculation** | Wrong fee, incorrect boundary, or fee charged after a failed transfer | Customer is overcharged | Revenue leakage or customer compensation/reconciliation issues | Financial-calculation defect |
| **P1** | **Beneficiary Validation** | Disabled/invalid beneficiary can receive funds, or beneficiary details are incorrect | Funds sent to unintended/invalid destination | Return/recovery costs and customer disputes | Integration/business-rule defect |
| **P1** | **AML / Sanctions / Fraud Controls** | Required screening is skipped, incorrectly approved, or incorrectly blocks legitimate transfers | Funds may be blocked or financial-crime exposure may occur | Regulatory, financial, and reputational consequences | Critical compliance/integration defect |
| **P1** | **Transaction State Management** | UI/API reports Success while backend is Pending/Failed/Reversed | Customer believes money was transferred when it was not | Complaints, reconciliation issues, operational cost | Loss of confidence in system correctness |
| **P1** | **Reversal / Recovery** | Failed transfer is not reversed after successful debit, or reversal happens twice | Money remains unavailable or is duplicated | Ledger imbalance / financial loss | Severe resilience and integration defect |
| **P1** | **Reconciliation** | CBS, switch, rail, and beneficiary records disagree and mismatch is not detected | Delayed or incorrect resolution of customer issue | Unresolved financial discrepancies | Indicates inadequate operational controls |
| **P2** | **Notification** | SMS is missing, duplicated, delayed, or contains incorrect transaction details | Customer lacks confirmation or receives misleading information | Support volume and communication issues | Functional/integration defect |
| **P2** | **Cutoff / Clearing Rules** | Transfer is processed in the wrong clearing window | Unexpected delay | Operational and customer-service impact | Business-rule defect |
| **P2** | **Input Validation** | Invalid amount/purpose data is accepted or malformed data reaches downstream systems | Failed or unexpected transfer behavior | Data-quality/security concerns | Preventable validation defect |


    

### 🎨 System Vulnerability Mapping (Risk Concentration)


    The feature's highest-risk boundary is the point where customer money is affected:

```text
    [Validation Layer]
    │
    ▼
    [Authentication Layer]
    │
    ▼
💰 ****CBS** Debit** <─────── [**FINANCIAL** **RISK** **BOUNDARY**]
    │
    ▼
    [Payment Rail]
    │
    ▼
    [Beneficiary Credit]
```

- **Before Debit:** Most failures are primarily simple **transaction rejections** (Low Operational Risk).
- **After Debit:** Failures escalate into **financial-integrity problems** requiring complex backend mechanisms:
- *Idempotency ➡️ State Recovery ➡️ Reconciliation ➡️ Automated Reversal*

---


## 👑 My Lead-Level Risk Priorities

### 🎯 Strategic Test Approach
**Testing implication:** I would allocate the majority of the test effort to **P0/P1 financial-integrity and security paths**, rather than distributing coverage evenly across the acceptance criteria.

### 🟢 P0 — Money & Security (Critical Integrity Tiers)
* **Duplicate Debit:** Prevention of multi-click transaction processing loops.
* **Debit + Unknown Outcome:** Asynchronous timeouts after successful CBS posting.
* **Concurrency / Balance:** Multi-device race conditions exhausting same available funds.
* **Limits:** Strict enforcement of transaction boundaries and cumulative ceilings.
* **Authentication Security:** OTP life-cycle binding and anti-replay defense frameworks.

### 🔴 P1 — Correctness & Compliance (Regulatory Tiers)
* **Fees Logic:** Accurate multi-segment fee calculations and failure state retention rules.
* **Beneficiary Validation:** Real-time routing state checks (Active, Frozen, Inactive accounts).
* **Compliance Filters:** Real-time AML, sanctions screening, and global watchlist intercepts.
* **State Management:** Strict transactional state-machine consistency within the database ledger.
* **Reversal & Reconciliation:** EOD reconciliation logs and automated rollback triggers.

### 🟡 P2 — Customer Experience (UX Tiers)
* **SMS Notifications:** Ensuring core transaction flow continues safely even during notification drops.


## Task 1.3 — Test Cases

Given the 12-case cap, I would optimize for risk coverage rather than one test per acceptance criterion. The suite deliberately prioritizes financial integrity, authentication, limits, idempotency, failure recovery, and critical integrations.

| ID | Test Case Title | Preconditions | Execution Steps | Expected Result |
|---|---|---|---|---|
| TC-01 | Verify Successful E2E Local Fund Transfer Via Instant Rail With Valid OTP And Ledger Posting Scope: Positive Core Integration Path | Active customer profile; Eligible source account; Registered active beneficiary; Sufficient balance; Fee config loaded. | 1. Select active source account. 2. Select active beneficiary. 3. Enter 5,000 and optional purpose note. 4. Click Submit and enter valid OTP. | UI: Confirmation screen with unique Txn Ref No displayed + Success SMS triggered. Backend/CBS: Balance debited by exactly 5,000 + fee. DB transaction status updated to Settled. |
| TC-02 | Verify Per Transaction Limit Enforcement At 10000 Exact Boundary Vs Immediate Rejection At 10000.01 Scope: Boundary Value Analysis (BVA) | Valid active customer & beneficiary; Available Balance = 25,000. | 1. Initiate transfer with exactly 10,000 + valid OTP. 2. Attempt a second transfer with 10,000.01. | Txn 1: Successfully executed, debited, and settled. Txn 2: Instantly blocked at UI/API Gateway layer; no OTP generated; zero ledger debit occurs. |
| TC-03 | Verify Daily Cumulative Limit Enforcement Allowing Cap At 20000 And Blocking Subsequent Transactions Scope: Cumulative Ledger Threshold Cap | Customer has already transferred 15,000 successfully today. | 1. Initiate transfer with exactly 5,000 + valid OTP. 2. Attempt another transfer with 0.01 immediately after. | Txn 1: Reaches the exact 20,000 daily cap and succeeds. Txn 2: Blocked by Limit Engine; explicit error message *Daily transfer limit exceeded* displayed. |
| TC-04 | Verify Transaction Initiation Fails When Principal Amount Plus Dynamic Fee Exceeds Available Source Balance Scope: Fee-Inclusive Insufficient Funds Handling | Available balance = 5,000; Transfer amount = 5,000; Applicable transfer fee > 0. | 1. Select source account. 2. Select beneficiary. 3. Enter amount 5,000 and click submit. | UI: Blocked before execution with *Insufficient funds* error. Backend/CBS: No funds blocked; account balance remains exactly 5,000 (Fee on top validation passed). |
| TC-05 | Verify OTP Lifecycle Security Enforcing Rejection On Invalid Expired And Reused Tokens To Prevent Replay Attacks Scope: Authentication & Session Token Invalidation | Valid transfer request initiated; OTP token generated. | 1. Enter incorrect OTP ➡️ Verify rejection. 2. Wait for expiration ➡️ Enter expired OTP ➡️ Verify rejection. 3. Enter valid OTP ➡️ Success. 4. Re-submit same OTP payload (Replay attack). | Steps 1, 2, 4: Rejected instantly; no debits occur. Step 3: Authorizes the intended transfer exactly once; OTP token is marked Used/Invalidated in DB. |
| TC-06 | Verify API Gateway Idempotency Intercepts Concurrent Duplicate Payloads To Guarantee Single Account Debit Scope: Concurrency & Message Deduplication | Valid transfer request payload ready; Same Idempotency-Key applied. | 1. Submit valid transfer request via API. 2. Concurrently fire an identical request payload using the exact same Idempotency-Key. | First Hit: Processes normally; generates unique Txn Ref No. Second Hit: Gateway intercepts key; returns cached response of first txn; strictly single ledger debit enforced. |
| TC-07 | Verify Resiliency State Transitions To Pending Recovery When Downstream Network Timeout Occurs After CBS Debit Scope: Distributed System Fault Tolerance & Reconciliation | Downstream timeout simulated; Sufficient balance available. | 1. Submit transfer and pass OTP. 2. Allow Core Banking debit to succeed. 3. Force network timeout/drop before Central Bank Switch response. | No duplicate transfer created; transaction record transitions to a recoverable state (Pending/Timeout); final state resolved via EOD Reconciliation. |
| TC-08 | Verify Automated Reversal Triggers To Restore Customer Funds Following Central Payment Rail Rejection Scope: Atomic Transactions & Rollback Reliability | Downstream rail failure simulated; Sufficient balance available. | 1. Submit transfer and pass OTP. 2. Allow CBS account debit to succeed. 3. Force central rail rejection/failure response. | UI: Shows transfer failure message. Backend: Triggers instant Auto-Reversal entry; money unblocked/credited back; database final state is Failed/Reversed. |
| TC-09 | Verify Race Condition Queueing Prevents Double Spending And Overdraft Under Simultaneous API Requests Scope: Multi-Threading Ledger Locking | Source account balance = 10,000; Concurrent execution engine active. | 1. Concurrently trigger three parallel transfer requests of 5,000 each at the exact same millisecond. | System queues requests; Txn 1 & 2 succeed (consuming 10,000). Txn 3 fails with insufficient funds. No account overdraft occurs. |
| TC-10 | Verify Fee Engine Applies Accurate Dynamic Fixed Vs Percentage Calculations Based On Customer Segments Scope: Dynamic Rules Pricing Engine | Multiple fee profiles loaded; Sufficient balance available. | 1. Execute txn under Profile A (Fixed fee). 2. Execute txn under Profile B (Percentage fee). 3. Force txn failure ➡️ Check fee status. | Correct fee calculated and posted to bank's internal revenue ledger. Failed transfers must reverse/release the fee entirely. |
| TC-11 | Verify Compliance Sanctions Engine Intercepts Transactions To Hold Suspended Or Reject Non Active Profiles Scope: AML Regulatory Controls & Verification | Beneficiary matrix configured with: ACTIVE, INACTIVE, BLACKLISTED. | 1. Transfer to ACTIVE beneficiary. 2. Transfer to INACTIVE beneficiary. 3. Transfer to BLACKLISTED (Sanctioned) account. | Active: Approved instantly (PASS). Inactive: Rejected at validation stage (REJECT). Blacklisted: Txn intercepted; status set to HOLD/SUSPENDED for manual compliance audit. |
| TC-12 | Verify Optional Purpose Note Sanitization And Character Length Limits Handle Special Coding Scripts Cleanly Scope: Security Validation & Input Boundary | Valid active customer & beneficiary; Sufficient available balance. | 1. Initiate valid transfer request. 2. In the optional purpose note field, input a maximum boundary text string embedded with SQL/XSS characters (e.g., Rent'; DROP TABLE...). | UI/API: Transaction completes successfully. Special characters are neutralized safely without system compilation distortion. Field maps safely into the DB log ledger. |

| Risk / AC                                   | Covered By                 |
| ------------------------------------------- | -------------------------- |
| Transfer initiation                         | TC-01, TC-11               |
| OTP authentication                          | TC-05                      |
| Fees                                        | TC-01, TC-04, TC-10        |
| Per-transaction limit                       | TC-02                      |
| Daily limit                                 | TC-03                      |
| Successful debit/reference/confirmation/SMS | TC-01, TC-12               |
| Failure handling                            | TC-04, TC-07, TC-08, TC-11 |
| Idempotency                                 | TC-06                      |
| Concurrency / double-spend                  | TC-09                      |
| Recovery / reversal                         | TC-07, TC-08               |
| Compliance / fraud                          | TC-11                      |
| Customer notification                       | TC-12                      |

### Financial Integrity

★★★★★  TC-06/07/08/09

Security & Access Control ★★★★★  TC-05/11

### Business Rules

★★★★☆   TC-02/03/04/10

### Core Functional Flow

★★★★☆   TC-01/12

The deliberate trade-off is not spending separate cases on low-risk UI permutations. The 12 cases concentrate on scenarios where a defect could cause financial loss, unauthorized transfer, incorrect customer balance, regulatory exposure, or unrecoverable transaction state.

### 1.4 — Coverage Decisions

**What did you deliberately choose not to cover, and why?**

I deliberately did not cover every possible UI, device, localization, or edge-case combination within the 12-case limit. For example, I did not create separate cases for every amount range, browser/device combination, or every possible purpose-note character/length variation.

I prioritized scenarios that could cause **financial loss, unauthorized transactions, incorrect balances, limit breaches, duplicate transfers, or unrecoverable transaction states**. Lower-risk variations can be covered through parameterization, exploratory testing, or broader regression testing.

---

**Which cases belong in an automated regression suite, and which stay manual? Why?**

**Automate:**

- TC-01 — Successful transfer
- TC-02 — Per-transaction limit
- TC-03 — Daily limit
- TC-04 — Balance validation
- TC-05 — **OTP** validation *(where **OTP** can be controlled in a test environment)*
- TC-06 — Idempotency / duplicate submission
- TC-07 — Debit + timeout recovery
- TC-08 — Rail failure and reversal
- TC-09 — Concurrent transfers
- TC-10 — Fee calculation

These are **repeatable, business-critical, regression-prone, and financially sensitive**. **API**/service-level automation should be preferred where possible, with a smaller number of critical end-to-end UI tests.

**Keep primarily manual/exploratory:**

- TC-11 — Beneficiary/compliance scenarios requiring configurable external decisions or human review
- TC-12 — Confirmation/**SMS** presentation and notification behavior

These may still have automated **API**/integration checks, but manual testing is valuable for **workflow behavior, external integrations, usability, and exploratory coverage**.

---

**How would you know this feature is safe to release?**

I would use explicit release criteria:

- All **P0/P1 test cases pass**, with no open critical/high defects affecting financial integrity or security.
- Source account debit, beneficiary credit, fees, limits, and transaction status are **financially consistent**.
- No duplicate debit is possible under retry/concurrent scenarios.
- Timeout, failure, reversal, and reconciliation scenarios have been successfully validated.
- **OTP**, beneficiary, fraud/compliance controls work as expected.
- Critical automated regression tests pass in CI.
- Transaction audit trails and monitoring are available for production.
- No unexplained **Pending/Unknown** transactions remain from testing.
- Business/Product/QA stakeholders approve the release based on the agreed acceptance and risk criteria.

**Release decision:** the feature is safe to release when the defined acceptance criteria are met, critical financial/security risks are closed or formally accepted, and the evidence demonstrates that both **successful and failure/recovery paths** behave correctly.