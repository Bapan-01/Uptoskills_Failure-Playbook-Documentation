# Failure Playbook: Fault Tolerance & Dead-Letter Queue (DLQ) Lifecycle

This playbook defines the architectural standards, operational procedures, and recovery paths for handling failures within distributed systems. It serves as a guide for engineers to design, monitor, and recover from failures, ensuring maximum system resilience and minimal data loss.

---

## 1. Introduction & Objectives

In a distributed microservices architecture, failures are inevitable. Networks partition, databases experience lock contention, downstream APIs suffer outages, and payloads can be malformed. Rather than attempting to prevent all failures, our systems must be designed to **survive** them.

### Objectives of this Playbook:
- Establish a uniform standard for classifying failures.
- Prevent cascading failures using the **Circuit Breaker** pattern.
- Smooth out transient spikes using **Exponential Backoff with Jitter**.
- Ensure zero data loss for unprocessable requests using a **Dead-Letter Queue (DLQ)**.
- Provide clear, step-by-step **Recovery Runbooks** for operators.

---

## 2. Failure Classification

When a service call or message processing fails, the system must immediately classify the failure to determine the appropriate response.

```
                  ┌───────────────────────────┐
                  │       Request Failed      │
                  └─────────────┬─────────────┘
                                │
                 Is the error likely temporary?
                                │
                ┌───────────────┴───────────────┐
                ▼                               ▼
        [ Yes: TRANSIENT ]              [ No: PERMANENT ]
        ┌────────────────┐              ┌────────────────┐
        │  Retry Allowed │              │ Do Not Retry   │
        └────────────────┘              └────────────────┘
```

### A. Transient Failures
Transient failures are temporary, self-correcting faults. They typically resolve quickly when the system is given time or when network congestion clears.
*   **Examples**:
    *   `503 Service Unavailable` or `504 Gateway Timeout` from a downstream API.
    *   Temporary network packet loss or socket timeouts.
    *   Database connection pool exhaustion or deadlock exceptions.
    *   Rate limiting (`429 Too Many Requests`) from third-party gateways.
*   **Handling Strategy**: Apply retry policies with exponential backoff and jitter. If the issue persists beyond the retry threshold, escalate to the DLQ.

### B. Permanent Failures
Permanent failures are static errors. No amount of retrying will make the request succeed without code changes, configuration updates, or payload modifications.
*   **Examples**:
    *   `400 Bad Request` due to validation errors (e.g., missing required fields).
    *   `401 Unauthorized` or `403 Forbidden` due to invalid credentials.
    *   `404 Not Found` when attempting to access a non-existent resource.
    *   `422 Unprocessable Entity` due to business logic violations.
*   **Handling Strategy**: **Do NOT retry.** Fail fast, return the error to the caller, or route the message directly to the DLQ if it is an asynchronous event.

### Failure Comparison Table

| Characteristic | Transient Failure | Permanent Failure |
| :--- | :--- | :--- |
| **Root Cause** | Network noise, load spikes, temporary outages | Code bugs, bad data, configuration errors |
| **Duration** | Seconds to minutes | Infinite (until resolved by intervention) |
| **Retry Action** | **Yes** (with backoff and jitter) | **No** (immediate fail-fast) |
| **Immediate Target** | Retry loop / Circuit Breaker | Client response / DLQ (for async tasks) |

---

## 3. Retry Policies & Mathematics

Retrying blindly can cause a **Thundering Herd** problem—where thousands of client instances retry at the exact same intervals, overloading the recovering downstream service and keeping it in a state of perpetual failure.

To prevent this, our retry policy uses **Exponential Backoff** combined with **Full Jitter**.

### A. The Mathematics of Backoff & Jitter

1.  **Exponential Backoff**:
    The delay before each retry attempt increases exponentially:
    \[
    t_{\text{backoff}} = t_{\text{base}} \times 2^{\text{attempt}}
    \]
    *Where \(t_{\text{base}}\) is the initial backoff interval, and \(\text{attempt}\) is the current retry count (starting at 0).*

2.  **Adding Jitter (Randomness)**:
    To spread out the retry requests evenly across time, we apply **Full Jitter**:
    \[
    t_{\text{retry}} = \text{Random}(0, \min(t_{\text{max}}, t_{\text{backoff}}))
    \]
    *Where \(t_{\text{max}}\) is the maximum allowable delay between retries.*

### B. Retry Timeline Example
Let's assume the following configuration:
*   \(t_{\text{base}}\) (Base Interval) = `1.0` second
*   \(t_{\text{max}}\) (Max Interval) = `30.0` seconds
*   \(\text{Max Retries}\) = `3`

Here is how the retry intervals are calculated:

| Attempt | Exponential Calculation (\(t_{\text{backoff}}\)) | Actual Delay Range (\(t_{\text{retry}}\)) | Example Actual Delay |
| :---: | :--- | :--- | :---: |
| **1** | \(1.0 \times 2^1 = 2.0\) seconds | `[0.0, 2.0]` seconds | **1.42s** |
| **2** | \(1.0 \times 2^2 = 4.0\) seconds | `[0.0, 4.0]` seconds | **3.11s** |
| **3** | \(1.0 \times 2^3 = 8.0\) seconds | `[0.0, 8.0]` seconds | **5.89s** |

> [!TIP]
> **Why Full Jitter?**
> By selecting a random value between `0` and the exponential limit, we guarantee that clients will be evenly distributed, flattening the traffic spike on the downstream service.

---

## 4. Circuit Breaker Integration

To prevent cascading failures across services, we place a **Circuit Breaker** between our service and any downstream dependency. 

```
          ┌──────────────────────────────────────────────┐
          │                                              │
          ▼                                              │ Success Rate
    ┌───────────┐           Failure Rate > Thresh       ┌┴──────────┐
    │  CLOSED   ├──────────────────────────────────────>│   OPEN    │
    └─────▲─────┘                                       └─────┬─────┘
          │                                                   │
          │               Limited Trial Successes             │ Cool-down
          └─────────────────── HALF-OPEN <────────────────────┘
                                            (Attempts trials)
```

### A. Circuit Breaker States
1.  **Closed**: 
    *   **Behavior**: Requests flow normally to the downstream service.
    *   **Action**: The system monitors failure rates and response times.
2.  **Open**:
    *   **Behavior**: The downstream service is failing. The Circuit Breaker intercepts all requests immediately.
    *   **Action**: Returns a local fallback response (e.g., cached data) or throws a fail-fast exception. It does not hit the network.
3.  **Half-Open**:
    *   **Behavior**: After a configured "cool-down period" (e.g., 60 seconds), the circuit enters this state.
    *   **Action**: Permits a small, controlled volume of traffic (e.g., 10 requests) to pass through.
    *   **Outcome**:
        *   If all trial requests succeed, the circuit returns to **Closed** (service recovered).
        *   If any trial request fails, the circuit immediately returns to **Open** (service still unhealthy).

### B. Interaction with the Retry Loop
*   If the Circuit Breaker is **Closed**, transient errors trigger the normal retry loop.
*   If the Circuit Breaker is **Open**, retries are bypassed. The system immediately executes the fallback or routes the request to the DLQ to avoid wasting resources.
*   If the Circuit Breaker is **Half-Open**, retries are disabled for the trial requests. If a trial request fails, it is treated as a permanent indicator of unhealthiness, and the circuit trips back to **Open**.

---

## 5. Dead-Letter Queue (DLQ) Architecture

When a request or message exhausted all retries or encountered an un-retryable permanent error, it is written to the **Dead-Letter Queue (DLQ)**. This prevents data loss and isolates the problematic messages.

### A. Ingestion Criteria
A message is routed to the DLQ under the following conditions:
1.  **Retry Exhaustion**: A transient failure persisted through all configured retries (e.g., `Retry Count == Max Retries`).
2.  **Permanent Processing Failure**: An unrecoverable logic error (e.g., schema mismatch, invalid JSON payload) occurred during parsing.
3.  **TTL Expiration**: The message remained in the main queue longer than its Time-To-Live (TTL) without being processed.

### B. Message Envelope Design
A message in the DLQ must not just contain the original payload. It **must** be wrapped in an envelope containing metadata to facilitate debugging and reprocessing.

```json
{
  "uuid": "d3b07384-d113-4a16-95ff-e380e227bb3c",
  "topic_origin": "payment-transactions",
  "ingested_at": "2026-06-28T18:52:00Z",
  "retry_count": 3,
  "failure_context": {
    "service": "payment-processor",
    "host": "pod-payment-5b9f77f-abc12",
    "error_code": "DOWNSTREAM_TIMEOUT",
    "error_message": "Read timed out after 5000ms from Stripe API",
    "exception_class": "java.net.SocketTimeoutException",
    "stack_trace": "java.net.SocketTimeoutException: Read timed out\n\tat java.net.SocketInputStream.socketRead0(Native Method)..."
  },
  "payload": {
    "transaction_id": "tx_99238471",
    "amount": 150.00,
    "currency": "USD",
    "user_id": "usr_772183"
  }
}
```

### C. Monitoring & Alerting
DLQs must be actively monitored. A growing DLQ is a leading indicator of an application bug or system degradation.
*   **Critical Metrics**:
    *   `DLQ_Message_Count`: Total number of messages in the DLQ. (Alert Threshold: `> 10` messages triggers a Warning; `> 100` triggers a Critical PagerDuty).
    *   `DLQ_Message_Age`: Age of the oldest message in the DLQ. (Alert Threshold: `> 4 hours` triggers an alert, indicating unprocessed failures).

---

## 6. Recovery Paths & Operational Runbooks

Once messages are in the DLQ, they must be processed. There are three primary recovery paths:

```
                            ┌────────────────────────┐
                            │    Messages in DLQ     │
                            └───────────┬────────────┘
                                        │
                                Analyze Failure
                                        │
                ┌───────────────────────┼───────────────────────┐
                ▼                       ▼                       ▼
        [ Path A: Auto ]        [ Path B: Manual ]      [ Path C: Archive ]
        ┌────────────────┐      ┌────────────────┐      ┌─────────────────┐
        │ Downstream Ok? │      │ Fix Code/Data  │      │ Unrecoverable?  │
        │ Re-drive       │      │ Edit & Re-drive│      │ Archive & Purge │
        └────────────────┘      └────────────────┘      └─────────────────┘
```

### Path A: Automated Re-drive (Service Recovery)
Used when the failure was caused by a temporary downstream outage that has since been resolved.
*   **Trigger**: Downstream service health checks return to `Green` and the Circuit Breaker is `Closed`.
*   **Action**: An automated script or consumer pulls messages from the DLQ and re-publishes them to the main queue.

### Path B: Manual Re-drive (Payload/Code Fix)
Used when the failure was caused by a bug in our code or a malformed payload.
*   **Trigger**: A software patch is deployed, or the payload is corrected.
*   **Action**: An operator uses the administrative console to edit the message payload (if necessary) and triggers a manual re-drive.

### Path C: Archive and Purge
Used when the message is corrupted, invalid, or obsolete, and cannot be processed.
*   **Trigger**: Business logic dictates the message is dead/unactionable.
*   **Action**: Archive the message payload to cold storage (e.g., S3 Glacier) for audit compliance, then purge it from the DLQ.

---

## 7. Step-by-Step Incident Response Runbooks

### Runbook 1: Downstream API Outage (e.g., Stripe Payment Gateway Outage)

> [!IMPORTANT]
> **Goal**: Protect the system from resource exhaustion and ensure payment requests are queued for reprocessing when Stripe recovers.

#### Phase 1: Detection
1.  **Alert**: Grafana dashboard shows a spike in `5xx` errors from the `payment-processor` service.
2.  **Alert**: PagerDuty triggers: "Payment Processor Circuit Breaker is OPEN."

#### Phase 2: Mitigation
1.  Verify the status of the downstream provider (e.g., check `status.stripe.com`).
2.  The Circuit Breaker automatically shifts to **Open** state, triggering the **Fallback Flow**:
    *   Users receive a polite message: *"We are experiencing payment processing delays. Your order has been queued and will be processed shortly."*
    *   The transaction request is immediately written to the `payment-dlq` with the error metadata `STRIPE_API_OUTAGE`.

#### Phase 3: Recovery
1.  Monitor downstream status until Stripe reports full recovery.
2.  The Circuit Breaker will transition to **Half-Open** and automatically close after successfully processing trial transactions.
3.  Execute the automated re-drive script to process the queued transactions in the DLQ:
    ```bash
    # Command to trigger automated re-drive of payment-dlq
    python scripts/redrive_queue.py \
      --source-queue payment-dlq \
      --target-queue payment-main \
      --batch-size 50 \
      --filter-error STRIPE_API_OUTAGE
    ```
4.  Monitor the `payment-processor` logs to ensure re-driven transactions are processed successfully.

---

### Runbook 2: Schema Mismatch / Parsing Failure

> [!WARNING]
> **Goal**: Resolve a permanent failure caused by a bad deployment or invalid payload schema without losing customer data.

#### Phase 1: Detection
1.  **Alert**: Slack channel `#alerts-dlq` receives a notification: *"CRITICAL: New messages in `order-processing-dlq` due to `PayloadParseException`."*

#### Phase 2: Diagnosis
1.  Log into the DLQ Admin Console or run the queue inspection tool:
    ```bash
    # Inspect the top 5 messages in the order-processing-dlq
    npm run queue:inspect -- --queue order-processing-dlq --limit 5
    ```
2.  Identify the error message:
    `JSON.parse error: Unexpected token x in JSON at position 45` or `Field 'user_id' cannot be null`.
3.  Check if this is a **code issue** (e.g., a bad release omitted a field) or a **client issue** (e.g., an external client sent bad data).

#### Phase 3: Resolution
1.  **If Code Issue**:
    *   Roll back the bad release to the previous stable version.
    *   Once the stable version is running, trigger a re-drive:
        ```bash
        npm run queue:redrive -- --source order-processing-dlq --target order-processing-main
        ```
2.  **If Client Issue (Malformed Payload)**:
    *   Contact the client/partner team regarding the malformed data.
    *   If the messages must be processed, download the payloads, patch them manually (or via script), and re-submit:
        ```bash
        # Export DLQ payloads to local file for editing
        npm run queue:export -- --queue order-processing-dlq --output ./scratch/malformed_orders.json
        
        # [Operator edits ./scratch/malformed_orders.json to fix user_id]
        
        # Re-import and submit to main queue
        npm run queue:import -- --file ./scratch/malformed_orders.json --target order-processing-main
        
        # Purge the processed messages from DLQ
        npm run queue:purge -- --queue order-processing-dlq
        ```
