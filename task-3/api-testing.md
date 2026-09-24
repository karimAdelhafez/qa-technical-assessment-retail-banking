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

### 3.1 Highest-Risk Scenarios
Because this endpoint actually moves money rather than just updating database fields, my testing focuses entirely on protecting customer funds and ledger boundaries:

* **Amount Precision:** I will pass extreme fractional values (like `1500.7582` or `0.000001`) to make sure the server rejects inputs that exceed standard currency decimal limits. This stops rounding issues that leak money.
* **Currency Verification:** I will try sending mismatch payloads (like a source account in `EGP` but sending the transfer value in `SAR`). The system must require an explicit, matching `fxQuoteId` to allow this, otherwise it should block it immediately to prevent unauthorized background conversions.
* **Concurrency and Race Conditions:** I will send duplicate requests at the exact same millisecond window. This checks if the backend uses proper **Pessimistic Database Record Locking**, safely returning a `402 Insufficient Funds` to the second request instead of letting an accidental double withdrawal slip through.
* **Limit Boundaries:** I will pass values right on and just over daily caps (like a transfer at `20,000.01` or a single request at `10,000.01`). The backend has to catch these boundary units and drop the transaction with a clear `422 Limit Exceeded` code.

### 3.2 Idempotency-Key Mechanics, Verification & Fault Recovery
* **Core Purpose:** The `Idempotency-Key` stops a customer from being charged twice. If a user double-clicks or retries a request because their mobile network drops, the API gateway ensures the backend executes the money transfer **exactly once**.
* **Verification Approach:** I will send a valid payment request that succeeds with a `201 Created`. I will immediately resend the exact same payload with the identical `Idempotency-Key`. The system passes if it skips the core accounting code entirely and instantly serves the cached receipt from the first run. Changing the payload while keeping the same key must fail with a `409 Conflict`.
* **Network Timeout & 503 Retries:**
  * **After a Network Timeout:** The actual transaction state is unknown. The mobile client must retry using the **identical Idempotency-Key**. If the server already completed the transaction before the network dropped, it returns the cached receipt safely. If it never reached the server, it processes it as a normal first-time transfer.
  * **After receiving a 503 Service Unavailable:** A `503` means the request failed at the gateway level *before* it ever reached the accounting core. Because the user's money was never touched, the mobile app should generate a **brand new Idempotency-Key** for its retry attempt to process the transfer normally.

### 3.3 End-to-End Money Movement Verification (Beyond HTTP 201)
Relying entirely on a surface-level `201 Created` or `201 Created` HTTP response code is a classic QA mistake. To guarantee the money actually moved securely, I would audit the backend infrastructure tracks:

* **Direct Database Queries:** I will check the ledger tables directly to verify that the source account's available and ledger balances dropped by the exact transfer amount plus fees, and that the beneficiary's rows reflect the correct matching credit.
* **Double-Entry Balance Checks:** I will check the central accounting core logs to verify that a balanced pair of debit and credit transaction lines were committed together to the general ledger.
* **Message Broker Verification:** I will trace the asynchronous event pipelines (like Apache Kafka or RabbitMQ logs) to ensure the payment service successfully published the correct JSON events to downstream Fraud Monitoring, AML, and Notification microservices.
* **End-of-Day Reconciliation:** I will match the final transaction parameters against the automated daily clearing file outputs (like ISO 20022 schemas or MT940 statements) to confirm the out-of-bank settlement data matches what the user initially authorized.
