# Task 3 — API Testing

This document outlines the test strategy and written analysis for our integration with the public Exchange Rates API (`://er-api.com`) and the upcoming outgoing payments endpoint.

---

## 🔍 1. Postman Test Strategy & Assertion Choices

The automated Postman suite runs **12 checks** across four base currencies (`USD`, `EUR`, `GBP`, `AED`) driven dynamically by `currency-data.json` to verify core integration requirements:

* **HTTP Status & Latency ([QE-01] & [QE-02]):** Asserts a strict `200 OK` status and a response time of `< 500ms`. Slow exchange rate updates freeze mobile client screens and cause user friction.
* **JSON Schema Integrity ([QE-03]):** Verifies that the JSON shape remains uncompromised and returns correct data types (e.g., `result` is a string, `rates` is an object, and timestamps are numbers).
* **Data Consistency ([QE-04]):** Cross-references the returned `base_code` against the active loop iteration parameter to prevent data cross-contamination during asynchronous runner batches.
* **Target Currency Boundaries ([QE-05], [QE-06], [QE-07]):** Verifies that the target currency (`EGP`) is present, parsed strictly as a numeric float, and is greater than zero (`rate > 0`). This prevents zero-division application crashes in conversion modules.

---

## 🔗 2. Rationale for Dynamic Chaining and Variable Capture

* **Captured Value:** `time_next_update_unix` (Next synchronization timestamp).
* **Why Chosen:** I captured this Unix timestamp from the first response payload, cached it into an environment variable (`captured_next_update`), and cross-referenced it inside the subsequent loop requests (**[QE-08]** & **[QE-09]**).
* **Testing Impact:** This acts as a protective gate against **Data Drift / Price Flipping**. If subsequent transactions within the same session pull different batch timestamps from an upstream service provider, it means multi-cluster sync has broken down, risking accounting leakage.

---

## 🚨 3. Critical Architectural Critique of the Negative Testing Path

During the negative loop requests (**[QE-10]** through **[QE-12]**), passing a malformed currency parameter (`INVALID`) reveals an unsafe design pattern from the upstream provider:

* **The Issue:** The API returns an **HTTP status code of `200 OK`** while burying the actual failure state inside the payload body (`"result": "error"`, `"error-type": "unsupported-code"`).
* **Architectural Flaw:** This approach violates standard REST practices (RFC 7231) and is highly unacceptable for production banking cores.
* **The Risk:** API Gateways, Firewalls, and Reverse Proxies monitor backend health using the HTTP status layer. When a bad or fraudulent request is wrapped inside a successful `200 OK` envelope, perimeter tools cannot trigger automated **Circuit Breakers** to isolate the system, forcing downstream databases to waste memory parsing junk data. The provider should return a strict `400 Bad Request` or `422 Unprocessable Entity`.

---

## 🛡️ 4. Enterprise Banking Security Strategy

To safely deploy this unauthenticated public integration into our production banking core, I would implement these three security layers:

1. **Authentication:** Upgrade from anonymous traffic to an encrypted **Mutual TLS (mTLS)** tunnel or secure IPsec VPN path with strict API key rotation.
2. **Rate Limiting:** Implement strict client-side caching gates and API gateway rate-throttling to insulate internal banking services from upstream DDoS vulnerabilities.
3. **Circuit Breaking:** Implement a fallback routine where an upstream network timeout or data failure automatically decouples the vendor server and serves the last valid **Cached Exchange Rate** saved within a safe 24-hour auditing window.

---

## 💸 Part B — Testing a Payments Endpoint

### 3.1 Highest-Risk Functional & Transactional Money Scenarios
Because this is a balance-altering money engine and not a basic CRUD database endpoint, my high-risk testing matrix focuses heavily on financial thresholds:

* **Amount Precision:** I will pass extreme fractional values (like `1500.7582` and `0.000001`) to ensure the server rejects inputs that exceed the currency's decimal exponent without causing rounding leakage.
* **Currency Contamination:** Test cases will pass mismatch payloads (e.g., source account in `EGP` but transaction amount in `SAR`). The system must require a valid `fxQuoteId` to allow cross-currency transfers, blocking unauthorized background conversions.
* **Concurrency Race Conditions:** I will execute rapid, parallel duplicate requests on the same source account within the exact same millisecond window. This verifies that the backend implements **Pessimistic Database Record Locking**, safely returning a `402 Insufficient Funds` to the second request instead of allowing an accidental double withdrawal.
* **Boundary Caps:** I will pass payloads exactly at and slightly above daily caps (e.g., trying to process a transfer at `20,000.01` or a single request at `10,000.01`). The backend must reliably catch these boundary lines and drop them with a deterministic `422 Limit Exceeded` response.

### 3.2 Idempotency-Key Mechanics, Verification & Fault Recovery
* **Core Purpose:** The `Idempotency-Key` is a safeguard managed at the API Gateway level. It guarantees that if a mobile client triggers duplicate clicks or retries a request due to a network drop, the backend will only process the transfer **exactly once**, preventing double-debiting.
* **Verification Approach:** I will send a valid payment request that returns a `201 Created`. I will immediately resend the identical request payload with the exact same `Idempotency-Key`. The server must bypass the ledger database completely and instantly return the cached receipt from the first transaction. Altering the payload while using the same key must return a `409 Conflict`.
* **Timeout & 503 Retries:**
  * **After a Network Timeout:** The transfer state is unknown. The client must safely retry using the **identical Idempotency-Key**. If the server already ran the transfer before the drop, it serves the cached receipt. If it never reached the server, it processes it safely as a fresh transaction.
  * **After a 503 Response:** A `503` proves the request choked at the gateway level *before* touching the business ledger. Since the money was never touched, the mobile application must generate a **brand new Idempotency-Key** for its retry attempt to avoid cross-locking backend databases.

### 3.3 End-to-End Money Movement Verification (Beyond HTTP 201)
Relying entirely on a surface-level `201 Created` HTTP response wrapper is a dangerous QA anti-pattern. To guarantee that customer funds actually shifted securely, I would implement automated backend checks:

* **Database Table Verification:** Query the account ledger rows directly to verify that the source account's available and ledger balances have dropped by the precise transfer amount plus calculated fees, while the beneficiary's rows reflect the matching credit.
* **Double-Entry Balance Audit:** Audit the central accounting core to ensure a balanced pair of debit and credit transaction lines were officially committed to the general journal.
* **Message Broker Tracing:** Inspect the event streaming pipelines (like Apache Kafka or RabbitMQ) to verify that the payment service published properly formatted JSON events to downstream Fraud Monitoring, AML screening, and Notification microservices.
* **Clearing House Reconciliation:** Verify the automated generation of daily clearing file outputs (like ISO 20022 message schemes or MT940 statement logs) to ensure that the out-of-bank transaction data matches the exact settlement details authorized by the initial API request.
