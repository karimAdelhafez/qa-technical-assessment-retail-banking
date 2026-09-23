# Task 2 — Exploratory Testing & Bug Reporting

## 🐛 2.1 Core Functional Defect Reports

### 🚨 [BUG-01]: App crashes with 404 error when submitting Transfer Funds with an empty amount field
* **Severity:** High 
* **Priority:** High

#### 🌐 Environment & Preconditions
* **URL:** `https://parasoft.com`
* **Preconditions:** User session is authenticated and active; source account has available funds; both source and destination account dropdown elements are fully populated.

#### 🛠️ Steps to Reproduce
1. Navigate to the main dashboard menu and click **"Transfer Funds"**.
2. Wait for the source and destination account dropdown menus to fully populate.
3. Leave the **"Amount"** input text box completely blank (Null token parameter payload).
4. Click the **"Transfer"** action button.
5. Open Browser Developer Tools (F12) and monitor the **Network Tab**.

#### 📉 Actual Result vs. Expected Result
* **Actual Result:** The interface completely crashes and displays a generic warning: *"An internal error has occurred and has been logged"*. The backend architecture encounters a routing layer breakdown, throwing a raw **`404 Not Found`** network status code on the execution endpoint.
* **Expected Result:** The application should catch the missing parameter via a client-side defensive validation layer before dispatching the payload. If it passes upstream, the server-side API should cleanly reject the payload with a structured **`400 Bad Request`** and explicit feedback: *"Amount field cannot be null or empty"*. It must never collapse the endpoint routing path.
* **Expectation Source:** Clean frontend and backend handling where the UI blocks empty submissions locally and the API (per RFC 7231 standards) gracefully flags payload anomalies rather than letting an unhandled server-side routing leak crash the app.

#### 💼 Business Justification & Systemic Impact
* **Severity Justification:** High system impact because an empty payload triggers an unhandled API routing engine breakdown under basic boundary conditions.
* **Priority Justification:** High business priority because cryptic server error feedback confuses users, driving an expensive spike in customer support center interactions.

#### 📸 Evidence Reference
* *Attached Workspace Snapshot:* `![Transfer Bug 404](evidence/bug_01_transfer_404.png)`

---

### 🚨 [BUG-02]: Applying for a loan with \$0 amount and \$0 down payment triggers a 500 Internal Server Error
* **Severity:** Critical
* **Priority:** High

#### 🌐 Environment & Preconditions
* **URL:** `https://parasoft.com`
* **Preconditions:** User session is authenticated; customer has an open checking or savings account baseline.

#### 🛠️ Steps to Reproduce
1. Navigate to the main dashboard menu and click **"Request Loan"**.
2. Type exactly `0` in the **"Loan Amount"** field.
3. Type exactly `0` in the **"Down Payment"** field.
4. Leave the account selection dropdown at its default loaded option.
5. Click the **"Apply Now"** action button.
6. Open Browser Developer Tools (F12) and monitor the **Network Tab response body**.

#### 📉 Actual Result vs. Expected Result
* **Actual Result:** The credit processing service encounters an uncaught runtime exception, returning a raw **`500 Internal Server Error`** network response code while the user interface renders a generic failure layout.
* **Expected Result:** The server-side credit engine should validate numerical ranges before running underwriting logic. It should reject the request cleanly with a **`400 Bad Request`** or an elegant credit rejection payload stating that the requested loan principal must be greater than zero.
* **Expectation Source:** Proper end-to-end input handling where the frontend rejects invalid zero-values on screen and the backend microservice enforces strict credit logic to catch unhandled mathematical exceptions before database processing.

#### 💼 Business Justification & Systemic Impact
* **Severity Justification:** High architecture impact because an uncaught server exception reveals an unstable, undefended backend runtime environment.
* **Priority Justification:** High business risk because microservice code crashes can cause resource leaks that directly impact core platform performance under heavy traffic loads.

#### 📸 Evidence Reference
* *Attached Workspace Snapshot:* `![Loan Bug 500](evidence/bug_02_loan_500.png)`

---

## 🧭 2.2 Exploratory Testing Approach & Roadmap

### 🕵️‍♂️ Initial Exploration Strategy: Why & Where?
Instead of random UI clicking, my methodology followed a strict **Risk-Based and Transactional Lifecycle Tour** targeting core security and financial validation boundaries:

* **Authentication & Authorization Audit:** Initiated the session by evaluating login session management, input field sanitization, and access token controls to guarantee unauthenticated profiles cannot bypass secure layout routing.
* **Core Financial Boundaries:** Target-tested the **Transfer Funds** and **Request Loan** endpoints because these modules process multi-parameter user payloads that directly alter active customer balances and execute downstream accounting routines.
* **Architectural Justification:** Focused on these specific components because any validation loop, authentication bypass, or unhandled null/zero values within financial microservices represents the highest systemic risk for ledger leakage, database truncation errors, or application downtime.


### 🗺️ Future Testing Roadmap: Next 2 Hours
If granted an additional 2 hours of exploration, I would systematically deploy my test coverage across the remaining modules specified in the assignment pack using these high-priority tactical tracks:

1. **Transfer Funds & Bill Pay End-to-End Execution:**
   * Execute extensive negative and boundary payload matrices on amount inputs (negative values, alphabetic scripts, trailing decimals, and over-limit inputs exceeding caps).
   * Verify end-to-end multi-account balance deduction states, recipient configurations, and check for concurrency race conditions (e.g., rapid multi-clicks bypassing UI locks before the API gateway processes the payload).
   * Test transactional synchronization to ensure user funds are blocked/reserved safely relative to downstream external clearing house acknowledgments.

2. **Advanced Loan Underwriting Logic (Request Loan):**
   * Stress-test the credit engine's internal mathematical logic by altering inputs across dependent fields to verify exact proportional limits (e.g., inputting `Down Payment > Loan Amount`).
   * Verify that illegal financial transaction vectors are gracefully trapped as handled business rejections rather than triggering unhandled server code exceptions.

3. **Account Integrity & Lifecycle Actions (Open New Account & Accounts Overview):**
   * Validate state consistency when creating a new account (Savings vs. Checking), checking if newly generated account IDs are instantly indexed in real-time balance calculations.
   * Verify session boundary enforcement across the account dashboard, ensuring parallel account queries cannot mix or leak data across different user sessions.

4. **Data Management, Search & Compliance (Find Transactions & Update Contact Info):**
   * Run heavy search parameter strings (by exact ID, specific amount range, or historical date boundaries) within the transaction ledger.
   * Validate that updating contact profiles (address, phone numbers) instantly synchronizes across downstream accounting engines and does not lock user session accounts during active fund transfers.
