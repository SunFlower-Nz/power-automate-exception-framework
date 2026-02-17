# Pattern 5: Circuit Breaker

## Overview

The **Circuit Breaker** pattern prevents a flow from repeatedly trying to access a failing system, which would waste resources and potentially make things worse. When failures reach a threshold, the circuit "opens" and fast-fails subsequent requests. In the Orchestrator, the pre-condition login check in Phase 3 is a practical example of this concept.

> See `Main.txt` Phase 3 (WQ-GenerateGuide login check) in [`examples/orchestrator-workqueue/`](../../examples/orchestrator-workqueue/)

## States

```
     ┌─────────┐    failures >= threshold    ┌──────────┐
     │  CLOSED  │ ─────────────────────────→ │   OPEN   │
     │ (normal) │                             │ (failing)│
     └────┬────┘                             └────┬────┘
          │                                       │
          │ success                                │ cooldown expires
          │                                       │
     ┌────┴────┐     success                 ┌────▼─────┐
     │  CLOSED  │ ←─────────────────────── │ HALF-OPEN │
     │ (normal) │     failure → back to OPEN │ (testing) │
     └─────────┘                             └──────────┘
```

| State | Behavior |
|-------|----------|
| **CLOSED** | Normal operation. Requests pass through. Failures are counted. |
| **OPEN** | Fast-fail all requests. No attempts to call the failing system. |
| **HALF-OPEN** | Allow ONE test request through. If it succeeds → CLOSED. If it fails → OPEN. |

## Real-World Example: Login Verification

In the Orchestrator, Phase 3 checks if the portal window is open before each item. If it's not, it kills all processes, re-logs in, then continues. This is a simplified circuit breaker — detecting system state before processing.

```
# ═══════ Pre-condition check (Circuit Breaker concept) ═══════
# Before processing each item, verify the system is available

IF (UIAutomation.IfWindow.IsNotOpenByTitleClass
        Title: $'''Guide Entry - Consultation*'''
        Class: $'''''') THEN
    # System is not available — "circuit is open"
    # Reset everything and re-establish connection
    CALL TerminateProcesses
    SET CurrentDF TO $'''DF0000-LoginPortal'''
    CALL StartDesktopFlow
    External.RunFlow FlowId: 'e053320f-...'
        @In_User: SystemUser
        @In_Password: SystemPassword
        @In_Environment: in_Environment
    CALL EndDesktopFlow
    # System re-established — "circuit closed"
END

# Now safe to process item
SET CurrentDF TO $'''DF0001-GenerateGuide'''
CALL StartDesktopFlow
External.RunFlow FlowId: '3f434cc8-...'
    @In_ItemData: WorkQueueItem.Input ...
CALL EndDesktopFlow
```

## Full Circuit Breaker Implementation (PAD Robin)

For advanced scenarios where you need a formal circuit breaker with state management:

### State Management via Config File

```
FUNCTION CircuitBreaker_Check GLOBAL
    /# ================================================
    Check if the circuit is open, half-open, or closed.
    State is persisted in a JSON file.
    ================================================#/

    # Read circuit state from file
    SET CircuitFile TO $'''%ExecFolder%\\circuit_state.json'''
    IF (File.IfFile.Exists File: CircuitFile) THEN
        File.ReadTextFromFile.ReadText File: CircuitFile
            Encoding: File.TextFileEncoding.UTF8 Content=> CircuitJson
        Variables.ConvertJsonToCustomObject Json: CircuitJson
            CustomObject=> CircuitState
    ELSE
        # Initialize: CLOSED state
        SET CircuitJson TO $'''{"State": "CLOSED", "FailureCount": 0,
            "Threshold": 5, "CooldownSeconds": 300,
            "LastFailureTime": "", "LastStateChange": ""}'''
        Variables.ConvertJsonToCustomObject Json: CircuitJson
            CustomObject=> CircuitState
    END

    SET CircuitCanProceed TO False

    IF CircuitState['State'] = $'''CLOSED''' THEN
        SET CircuitCanProceed TO True

    ELSE IF CircuitState['State'] = $'''OPEN''' THEN
        # Check if cooldown has expired
        DateTime.GetCurrentDateTime.Local CurrentDateTime=> Now
        # Compare with LastFailureTime
        # If enough time passed → transition to HALF_OPEN
        SET CircuitCanProceed TO False

        # ==== Write log ====
        SET SubFlowName TO $'''CircuitBreaker_Check'''
        SET TaskLine TO 0
        SET LogText TO $'''Circuit OPEN: fast-failing. System unavailable.'''
        CALL WriteLog

    ELSE IF CircuitState['State'] = $'''HALF_OPEN''' THEN
        # Allow one test request
        SET CircuitCanProceed TO True

        SET SubFlowName TO $'''CircuitBreaker_Check'''
        SET TaskLine TO 0
        SET LogText TO $'''Circuit HALF_OPEN: testing with 1 request'''
        CALL WriteLog
    END
END FUNCTION
```

### Record Success / Failure

```
FUNCTION CircuitBreaker_RecordSuccess GLOBAL
    IF CircuitState['State'] = $'''HALF_OPEN''' THEN
        # Test succeeded — close the circuit
        SET CircuitJson TO $'''{"State": "CLOSED", "FailureCount": 0,
            "Threshold": 5, "CooldownSeconds": 300,
            "LastFailureTime": "", "LastStateChange": "%Now%"}'''
        File.WriteText File: CircuitFile TextToWrite: CircuitJson
            IfFileExists: File.IfFileExists.Overwrite

        SET SubFlowName TO $'''CircuitBreaker'''
        SET LogText TO $'''Circuit CLOSED: system recovered'''
        CALL WriteLog
    END

    # Reset failure count
    SET CircuitState['FailureCount'] TO 0
END FUNCTION
```

```
FUNCTION CircuitBreaker_RecordFailure GLOBAL
    SET FailCount TO CircuitState['FailureCount'] + 1

    IF CircuitState['State'] = $'''HALF_OPEN''' THEN
        # Test failed — re-open circuit
        SET CircuitJson TO $'''{"State": "OPEN", "FailureCount": %FailCount%,
            "Threshold": 5, "CooldownSeconds": 300,
            "LastFailureTime": "%Now%", "LastStateChange": "%Now%"}'''
        File.WriteText File: CircuitFile TextToWrite: CircuitJson
            IfFileExists: File.IfFileExists.Overwrite

        SET LogText TO $'''Circuit OPEN: test failed. %ErrorMessage%'''
        CALL WriteLog
        RETURN
    END

    IF FailCount >= CircuitState['Threshold'] THEN
        # Threshold exceeded — open circuit
        SET CircuitJson TO $'''{"State": "OPEN", "FailureCount": %FailCount%,
            "Threshold": 5, "CooldownSeconds": 300,
            "LastFailureTime": "%Now%", "LastStateChange": "%Now%"}'''
        File.WriteText File: CircuitFile TextToWrite: CircuitJson
            IfFileExists: File.IfFileExists.Overwrite

        SET LogText TO $'''Circuit OPEN: %FailCount% consecutive failures'''
        CALL WriteLog
    ELSE
        # Below threshold — increment count
        SET LogText TO $'''Failure %FailCount%/%CircuitState['Threshold']%'''
        CALL WriteLog
    END
END FUNCTION
```

### Using the Circuit Breaker in a Flow

```
# Before calling an external system
CALL CircuitBreaker_Check

IF CircuitCanProceed = False THEN
    # Circuit is open — skip and queue for later
    SET SubFlowName TO $'''Main'''
    SET LogText TO $'''Skipping: circuit breaker OPEN'''
    CALL WriteLog

    # Mark item for retry later
    WorkQueues.UpdateWorkQueueItem.UpdateWorkQueueItem
        WorkQueueItem: WorkQueueItem
        Status: WorkQueues.WorkQueueItemStatus.GenericException
        ProcessingNotes: $'''Circuit breaker OPEN - system unavailable'''
ELSE
    # Circuit is closed or half-open — try the action
    External.RunFlow FlowId: '...'
        @In_ItemData: WorkQueueItem.Input ...
    ON ERROR
        CALL CircuitBreaker_RecordFailure
    END

    # If successful
    CALL CircuitBreaker_RecordSuccess
    WorkQueues.UpdateWorkQueueItem.UpdateWorkQueueItem
        WorkQueueItem: WorkQueueItem
        Status: WorkQueues.WorkQueueItemStatus.Processed
        ProcessingNotes: $'''Processed successfully'''
END
```

## Configuration Recommendations

| System Type | Threshold | Cooldown | Notes |
|-------------|-----------|----------|-------|
| Web portal | 3 failures | 120s | Re-login may fix it |
| External API | 5 failures | 300s | Less control over recovery |
| Legacy desktop app | 3 failures | 120s | May need process restart |
| File server | 5 failures | 180s | May be maintenance window |

## When to Use vs. Simple Pre-Condition

| Scenario | Use Circuit Breaker | Use Pre-Condition Check |
|----------|:---:|:---:|
| Web portal that disconnects | | Yes (check window exists) |
| External API with rate limits | Yes | |
| System with scheduled downtime | Yes | |
| Application that needs restart | | Yes (kill + relaunch) |
| Multiple flows hitting same system | Yes (shared state) | |

---

*Next: [Pattern 6 — Alerting & Monitoring](../06-alerting-monitoring/)*
