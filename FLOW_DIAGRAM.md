# Failure Lifecycle Flow Diagram

This document provides a visual representation of the end-to-end request lifecycle, detailing how failures are classified, how the retry mechanism (with Exponential Backoff & Jitter) operates, how the Circuit Breaker interacts with the flow, and how permanently failed requests are routed to and recovered from the Dead-Letter Queue (DLQ).

---

## 📊 End-to-End Failure Lifecycle

```mermaid
flowchart TD
    %% Styling
    classDef startEnd fill:#1F2937,stroke:#111827,stroke-width:2px,color:#fff;
    classDef process fill:#3B82F6,stroke:#1D4ED8,stroke-width:2px,color:#fff;
    classDef decision fill:#F59E0B,stroke:#D97706,stroke-width:2px,color:#fff;
    classDef failure fill:#EF4444,stroke:#B91C1C,stroke-width:2px,color:#fff;
    classDef success fill:#10B981,stroke:#047857,stroke-width:2px,color:#fff;
    classDef storage fill:#8B5CF6,stroke:#6D28D9,stroke-width:2px,color:#fff;

    Start([Start: Client Request]) --> CB_Check{Circuit Breaker State?}
    
    %% Circuit Breaker Decision
    CB_Check -- Open ----> Fallback[Execute Fallback / Fail-Fast]
    CB_Check -- Closed / Half-Open --> SendRequest[Send Request to Target Service]
    
    %% Request Outcome
    SendRequest --> ResponseCheck{Is Request Successful?}
    ResponseCheck -- Yes ----> SuccessResult([Success: Return 200 OK / Acknowledge])
    ResponseCheck -- No --> ClassifyFailure{Classify Failure Type}
    
    %% Failure Classification
    ClassifyFailure -- Permanent <br> e.g., 400 Bad Request, 403 Forbidden --> PermanentFail([Permanent Failure: Stop & Return Error])
    ClassifyFailure -- Transient <br> e.g., 503 Service Unavailable, Timeout --> RetryCheck{Retry Count < Max Retries?} 
    
    %% Retry Loop
    RetryCheck -- Yes --> CalcBackoff[Calculate Backoff Delay <br> Exponential Backoff + Jitter]
    CalcBackoff --> WaitDelay[Wait for Calculated Delay]
    WaitDelay --> RetryIncrement[Increment Retry Count]
    RetryIncrement --> SendRequest
    
    %% DLQ Route
    RetryCheck -- No --> RouteDLQ[Route Message/Request to DLQ]
    RouteDLQ --> AddMetadata[Attach Error Metadata <br> Timestamp, Attempt Count, Exception]
    AddMetadata --> AlertSystem[Trigger Alerts <br> PagerDuty, Slack, Prometheus]
    
    %% DLQ Operations & Recovery
    AlertSystem --> DLQStorage[(Dead-Letter Queue Storage)]
    DLQStorage --> OperatorInspect[Operator/System Inspects DLQ]
    
    OperatorInspect --> RecoveryChoice{Recovery Decision}
    RecoveryChoice -- Re-drive / Re-process --> ResolveRoot[Resolve Root Cause <br> e.g., Fix Downstream Service]
    ResolveRoot --> RedriveMain[Re-drive Messages back to Main Queue / Retry Request]
    RedriveMain --> CB_Check
    
    RecoveryChoice -- Purge / Discard --> ArchiveS3[Archive to Cold Storage <br> e.g., AWS S3 / Glacier]
    ArchiveS3 --> PurgeQueue([Purge Message from DLQ])
```

---

## 🔍 Key Stages Explained

### 1. Circuit Breaker Check
Before sending the request, the client or gateway checks the state of the **Circuit Breaker** protecting the target service.
- **Closed**: Requests flow normally.
- **Open**: The service is known to be failing; the request is intercepted immediately to prevent overloading the downstream system (Fail-Fast).
- **Half-Open**: A limited number of trial requests are allowed through to see if the service has recovered.

### 2. Failure Classification
When a request fails, it is immediately classified:
- **Transient Failures**: Intermittent issues (e.g., rate limits, network packet loss, temporary database locks). These are suitable for retries.
- **Permanent Failures**: Logic or input errors (e.g., validation errors, missing resource, authentication failure). Retrying these would yield the same result and waste resources.

### 3. Retry Loop with Exponential Backoff and Jitter
If the failure is transient, the system delays the next attempt. The delay grows exponentially to allow the downstream system time to recover, and "Jitter" (randomness) is added to prevent **Thundering Herd** scenarios (where all retrying clients hit the server at the exact same millisecond).

### 4. Dead-Letter Queue (DLQ) Routing
If all retries are exhausted without success, the request is marked as dead. Instead of silently dropping it, the system moves the request payload along with comprehensive error metadata (stack traces, attempt counts, timestamps) into the **Dead-Letter Queue (DLQ)**.

### 5. Recovery & Re-drive
Once in the DLQ, messages remain isolated until an operator or automated script takes action:
- **Resolve Root Cause**: The underlying issue is fixed (e.g., database scaled up, bug patched).
- **Re-drive**: Messages are re-routed back into the main processing queue to be processed successfully.
- **Archive**: Unprocessable messages are stored in cold storage for post-mortem analysis and purged from the DLQ.
