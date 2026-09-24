# Task 3 (Part A) — API Testing

This document presents the technical analysis of the upstream Exchange Rates API (`://er-api.com`) integrated into the international fund transfer sub-system, detailing the validation mechanics and architectural safeguards required for a production retail banking core.

---

## 🔍 1. Postman Test Strategy & Assertion Choices

The Postman test suite executes exactly **12 automated contract and data-type verification checks** across 4 currencies (`USD`, `EUR`, `GBP`, `AED`) driven by `currency-data.json` to guarantee that data aligns with banking ledger requirements:

* **HTTP Status & Performance Core ([QE-01] & [QE-02]):** Asserts strict `200 OK` status and enforces a **< 500ms latency threshold**. In retail banking, slow FX rate responses freeze user checkout screens, causing immediate session friction and abandonment.
* **JSON Schema Shape Integrity ([QE-03]):** Implements absolute schema type checking. It guarantees that the response envelope format remains uncompromised and wraps correct types (e.g., `result` is a string, `rates` is an object, and timestamps are numbers).
* **Data Parameter Consistency ([QE-04]):** Cross-references the response `base_code` against the active loop iteration parameter (`USD`, `EUR`, etc.) to prevent thread cross-contamination during bulk asynchronous querying.
* **Target Presence, Numeric Type, and positive Boundaries ([QE-05], [QE-06], [QE-07]):** Verifies that the recipient target settlement currency (**`EGP`**) is present, parsed strictly as a float/numeric type, and is **strictly greater than zero (`rate > 0`)**. This block prevents zero-division runtime crashes in dynamic conversion calculators and blocks catastrophic negative scalar valuations.

---

## 🔗 2. Rationale for Dynamic Chaining and Variable Capture

* **Captured Parameter:** `time_next_update_unix` (Next synchronization batch timestamp).
* **Engineering Rationale:** In a core banking implementation, international transfers must map to a strict cryptographic and chronological window. I captured this unix timestamp from the first response, stored it into the environment scope (`captured_next_update`), and cross-referenced it in the subsequent request (**[QE-08]** & **[QE-09]**). 
* **Business Resiliency Impact:** This data consistency gate prevents **Data Drift / Price Flipping**. If sequential transactions within the same session pull differing baseline updating timestamps from an upstream provider, it indicates an unstable multi-cluster sync state, which can cause severe financial accounting leakage.

---

## 🚨 3. Critical Architectural Critique of the Negative Testing Path

During the negative execution loop (**[QE-10]** through **[QE-12]**), passing a malformed currency parameter (`INVALID`) reveals a highly dangerous design pattern from the upstream provider:

* **The Trap:** The API returns an **HTTP status code of `200 OK`** while burying the actual failure state inside the JSON payload body (`"result": "error"`, `"error-type": "unsupported-code"`).
* **Architectural Flaw:** **This design is highly unacceptable for enterprise banking architectures.** It violates standard REST standards (RFC 7231). 
* **The Risk:** API Gateways, Edge Firewalls, and Reverse Proxies monitor backend infrastructure health using the HTTP status layer. When a broken or malicious payload is wrapped inside a successful `200 OK`, perimeter tools cannot trigger automated **Circuit Breakers** or flag rapid attack signatures, forcing downstream core computing resources to actively waste memory parsing fraudulent or corrupted execution payloads. The provider should return a strict **`400 Bad Request`** or **`422 Unprocessable Entity`**.

---

## 🛡️ 4. Enterprise Banking Security Strategy

To safely deploy this public, no-key integration into a real production core environment, I would mandate the implementation of these three missing security and resilience layers:

1. **Authentication & Transport Layer Security:** Upgrade from unauthenticated channels to an enforced **Mutual TLS (mTLS)** architecture using client certificates, or wrap traffic within an encrypted IPsec VPN tunnel with strict API key rotation.
2. **Perimeter Throttling & Rate Limiting:** Enforce strict client-side caching limits and API gateway rate-throttling to insulate the banking backend from upstream DDoS vulnerabilities.
3. **Graceful Degradation Circuit Breaking:** Implement a fallback logic routine where a server `5xx/4xx` or internal payload failure automatically decouples the upstream server and serves the last valid **Cached Exchange Rate** fetched within a safe 24-hour auditing window.
---

## 💸 5. Part B — Payments Endpoint Strategic Analysis (`POST /api/v1/payments`)

### 3.1 Highest-Risk Functional & Transactional Money Scenarios
This is a critical balance-altering transaction engine, not an ordinary CRUD database endpoint. My high-risk testing matrix isolates parameters to prevent financial leakage and logic failures:

* **Amount Object Floating-Point Precision:** I will pass extreme fractional values (e.g., `1500.7582` and `0.000001`) to verify that the server strictly rejects inputs exceeding the currency's standard exponent or cleanly handles rounding logic without causing systematic fractional currency drops.
* **Currency Cross-Contamination & Validation:** Test cases will pass mismatch payloads (e.g., `sourceAccountId` configured in EGP, but `amount.currency` sent as `SAR`). I will verify that the server forces a hard block if a valid `fxQuoteId` is missing, preventing illegal backend currency conversions.
* **Concurrency & Race Condition Multi-Debits:** I will execute rapid, parallel twin requests on the same source account within the exact same millisecond window. I am testing to ensure that the backend implements robust **Pessimistic Database Record Locking**, gracefully returning a `402 Insufficient Funds` or handled business rejection to the second hit rather than allowing a double withdrawal or account overdraft.
* **Limit Matrix Truncation Boundaries:** I will pass payloads triggering transaction counts exactly at, and one minor decimal unit above, the active daily limits (e.g., trying to process a transfer at `20,000.01` or a single request at `10,000.01`). The backend must reliably catch these boundary vectors and drop them with a deterministic `422 Limit Exceeded` response.

### 3.2 Idempotency-Key Mechanics, Verification & Fault Recovery
* **Core Purpose:** The `Idempotency-Key` serves as an active transaction safeguard on the API Gateway. It guarantees that if a mobile client triggers duplicate clicks or retries a request due to volatile network drops, the backend will only execute the underlying financial transfer **exactly once**, preventing double-debiting customer funds.
* **Verification Approach:** I will send an initial valid payment request which successfully returns a `201 Created`. I will immediately resend the identical payload with the exact same `Idempotency-Key`. The system passes if the server completely bypasses the core accounting code, touches zero balance tables, and instantly returns the cached response of the first transaction. If a client alters the transaction data while using the same key, the gateway must flag payload tampering and return a `409 Conflict` state.
* **Network Timeout & 503 Retry Handlers:** 
  * **After a Network Timeout:** The transaction outcome is temporarily unknown. The mobile client must safely retry using the **identical Idempotency-Key**. If the server already processed the original hit before the timeout, it will gracefully serve the cached receipt. If it never reached the server, it will safely execute it as a fresh transaction.
  * **After a 503 Service Unavailable Response:** A `503` proves that the gateway choked or was overloaded *before* parsing the request into the core business ledger. Because the customer's money was never touched, the client application should generate a **brand new Idempotency-Key** for its retry attempt to process the transfer safely without ledger cross-locking.

### 3.3 End-to-End Money Movement Verification (Beyond HTTP 201)
Relying entirely on a surface-level HTTP response wrapper is a severe QA anti-pattern. To guarantee that customer funds actually shifted securely across distributed boundaries, I would implement automated end-to-end database, ledger, and ledger reconciliation checks:

* **Atomic Database Table Verification:** Direct row-level queries on the account ledger tables to verify that the source account's *available balance* and *ledger balance* have dropped by exactly the transfer value plus calculated fees, while the destination beneficiary's account rows reflect the precise matching credit.
* **Double-Entry General Ledger Balance Audit:** Verify that a balanced pair of balancing debit and credit transaction lines were officially committed to the central core accounting repository (Debit Source Account Asset ➡️ Credit Intermediate Settlement/Beneficiary Liability account).
* **Distributed Message Broker Inspection:** Trace the asynchronous event streaming pipeline (e.g., Apache Kafka or RabbitMQ logs) to ensure that the payment microservice fired highly formatted transactional event blocks containing the correct payload signatures to downstream Fraud Monitoring, AML screening, and Notification microservices.
* **End-of-Day Clearing House Reconciliation Logs:** Verify the automated extraction and generation of daily clearing file outputs (such as ISO 20022 message schemes or MT940 statement records) to ensure that the out-of-bank transaction matches the exact settlement details authorized by the initial API request payload.
