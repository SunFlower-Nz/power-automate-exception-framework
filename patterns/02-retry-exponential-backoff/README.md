# Pattern 2: Retry with Exponential Backoff

## Overview

Handles **transient failures** — temporary errors that resolve on their own (API timeouts, network issues, locked files, busy applications). In the Orchestrator framework, Work Queues handle retry automatically via `GenericException` status. This pattern covers both approaches: **native WQ retry** (recommended) and **manual retry** for actions outside Work Queues.

> See `UpdateQueueItemError` subflow in [`examples/orchestrator-workqueue/`](../../examples/orchestrator-workqueue/) for the production implementation.

## When to Use

- API calls returning 429 (Too Many Requests) or 503 (Service Unavailable)
- Web page loading timeout
- File locked by another process
- Application not responding temporarily
- Child Desktop Flow transient failures

## Two Approaches

### Approach 1: Work Queue Native Retry (Recommended)

When using Work Queues, you **don't write retry logic**. You mark the item as `GenericException` and the Work Queue handles the retry automatically based on the `Max Retries` configured in the queue.

```
┌─────────────────────────────────────────────────────┐
│    Work Queue Native Retry (configured in portal)    │
├─────────────────────────────────────────────────────┤
│                                                       │
│  Item dequeued → Processing                           │
│       │                                               │
│       ├─ Success → UpdateWorkQueueItem (Processed)    │
│       │                                               │
│       └─ Error → UpdateWorkQueueItem                  │
│                   (GenericException)                   │
│                           │                           │
│              ┌────────────▼─────────────┐             │
│              │  Work Queue decides:      │             │
│              │  Attempts < MaxRetries?   │             │
│              │  ├─ YES → Requeue item    │             │
│              │  └─ NO  → Mark as Failed  │             │
│              └──────────────────────────┘             │
│                                                       │
└─────────────────────────────────────────────────────┘
```

### Approach 2: Manual Retry with Backoff

For actions **outside** Work Queues (API calls, web scraping, login attempts), implement retry manually with exponential delay.

```
┌────────────────────────────────────────────┐
│           Manual Retry Logic                │
├────────────────────────────────────────────┤
│                                            │
│  MaxRetries = 3  (from config)             │
│  CurrentRetry = 0                          │
│                                            │
│  LOOP WHILE CurrentRetry < MaxRetries      │
│  │                                         │
│  │  Action with ON ERROR                   │
│  │  ├─ Success → EXIT LOOP                │
│  │  └─ Error                               │
│  │       ├─ CurrentRetry++                 │
│  │       ├─ WAIT (2^retry) seconds         │
│  │       └─ Log retry attempt              │
│  │                                         │
│  END LOOP                                  │
│                                            │
│  IF still failed → escalate                │
│                                            │
└────────────────────────────────────────────┘
```

## Implementation

### Work Queue Native Retry (Production Code)

This is the real `UpdateQueueItemError` subflow from the Orchestrator. When a child flow fails, the item is marked with `GenericException` and the Work Queue decides whether to retry.

```
FUNCTION UpdateQueueItemError GLOBAL
    /# ================================================
    Signal error on the Work Queue item. The Work Queue manages
    retry automatically based on the configured Max Retries.
    If the item hasn't reached Max Retries, it goes back to "Queued".
    If it has, it stays as "Failed" permanently.
    ================================================#/
    IF (DesktopFlowFailed = True AND UpdateQueueError = 1) = True THEN
        WorkQueues.UpdateWorkQueueItem.UpdateWorkQueueItem
            WorkQueueItem: WorkQueueItem
            Status: WorkQueues.WorkQueueItemStatus.GenericException
            ProcessingNotes: $'''Error: %ErrorMessage%'''
            ProcessingResult: ErrorBody

        # ==== Write log ====
        SET SubFlowName TO $'''UpdateQueueItemError'''
        SET TaskLine TO 5
        SET LogText TO $'''Queue item marked with error
            (automatic retry by Work Queue): %ErrorMessage%'''
        CALL WriteLog
    END
END FUNCTION
```

**Work Queue configuration (portal):**

| Work Queue | Max Retries | Effect |
|------------|-------------|--------|
| `WQ-MoveSpreadsheet` | 3 | Item retried up to 3x, then `Failed` |
| `WQ-GenerateGuide` | 3 | Item retried up to 3x, then `Failed` |
| `WQ-GenerateReport` | 3 | Item retried up to 3x, then `Failed` |
| `WQ-Execution` | 0 | No retry (execution tracking only) |
| `WQ-Errors` | 0 | No retry (error logging only) |

### The `UpdateQueueError` Flag

This flag controls whether the error handler should mark a Work Queue item as failed:

```
# Before dequeue — no item to update yet
SET UpdateQueueError TO 0

# After successful dequeue — item exists, should be updated on error
WorkQueues.DequeueWorkQueueItem.DequeueWorkQueueItem
    QueueName: $'''WQ-GenerateGuide'''
    WorkQueueItem=> WorkQueueItem
ON ERROR
    SET UpdateQueueError TO 0    # Empty queue, no item to update
    CALL EndDesktopFlow
    EXIT LOOP
END

SET UpdateQueueError TO 1        # Item exists — update on error
SET QueueItemId TO WorkQueueItem.Id
```

### Manual Retry (for actions outside Work Queues)

For operations like login, web scraping, or API calls that aren't managed by a Work Queue:

```
# Read max retries from config JSON
SET MaxRetries TO ConfigContents['Retries']['Max_Retry_Count']

SET RetryCount TO 0
SET ActionSuccess TO False

LOOP WHILE RetryCount < MaxRetries AND ActionSuccess = False

    SET RetryCount TO RetryCount + 1

    # Action with ON ERROR per-action handler
    Web.InvokeWebService.InvokeWebService
        Url: $'''https://api.example.com/data'''
        Method: Web.Method.Get
        Timeout: 30
        Response=> ApiResponse
    ON ERROR
        # ==== Write log ====
        SET SubFlowName TO $'''Main'''
        SET TaskLine TO 45
        SET LogText TO $'''Attempt %RetryCount%/%MaxRetries%
            failed: %LastError.Message%. Waiting...'''
        CALL WriteLog

        IF RetryCount < MaxRetries THEN
            # Exponential backoff: 2^retry seconds (2s, 4s, 8s)
            SET Delay TO 2 * (2 ^ RetryCount)
            WAIT Delay
        END
        GOTO NextIteration
    END

    # If we get here, action succeeded
    SET ActionSuccess TO True
    SET SubFlowName TO $'''Main'''
    SET TaskLine TO 55
    SET LogText TO $'''Action succeeded on attempt %RetryCount%'''
    CALL WriteLog

    LABEL NextIteration
END LOOP

# Check if ultimately failed
IF ActionSuccess = False THEN
    SET Status TO $'''Error'''
    SET Observation TO $'''Action failed after %MaxRetries% attempts'''
END
```

### Per-Action Retry with `ON ERROR REPEAT`

PAD also has a built-in `ON ERROR REPEAT N TIMES WAIT S` for simple cases:

```
# Screenshot with automatic retry (1 attempt, 2 second wait)
Workstation.TakeScreenshot.TakeScreenshotOfPrimaryScreenAndSaveToFile
    File: EvidenceFile
    ImageFormat: System.ImageFormat.Jpg
ON ERROR REPEAT 1 TIMES WAIT 2
ON ERROR
    # If retry also fails, silently continue
END
```

## Delay Calculation

| Retry # | Formula | Delay | Total Elapsed |
|---------|---------|-------|---------------|
| 1 | 2^1 x 2 | 4s | 4s |
| 2 | 2^2 x 2 | 8s | 12s |
| 3 | 2^3 x 2 | 16s | 28s |

## Configurable Parameters

The max retries value comes from the config JSON passed by the Cloud Flow:

```json
{
    "Retries": {
        "Max_Retry_Count": 3
    }
}
```

Read in PAD:
```
Variables.ConvertJsonToCustomObject Json: in_ConfigJson CustomObject=> ConfigContents
SET MaxRetries TO ConfigContents['Retries']['Max_Retry_Count']
```

## When NOT to Retry

| Error Type | Should Retry? | Why |
|-----------|:---:|-----|
| Element not found (UI) | No | Screen layout changed, won't fix itself |
| Invalid credentials | No | Wrong credentials won't magically fix |
| File not found | No | Resource doesn't exist |
| File locked | Yes | Another process will release |
| Failed to run flow | Yes | Child flow transient failure |
| Web timeout | Yes | Network or load issue |
| Application not responding | Yes | May recover after wait |
| Window not found | Yes | Application may still be loading |

## Work Queue vs Manual: Decision Guide

| Scenario | Use Work Queue Retry | Use Manual Retry |
|----------|:---:|:---:|
| Processing batch items (invoices, files) | Yes | |
| Login to a portal | | Yes |
| Single API call | | Yes |
| Screenshot capture | | Yes (`ON ERROR REPEAT`) |
| Child Desktop Flow | Yes | |
| Page load wait | | Yes |

---

*Next: [Pattern 3 — Work Queue Error Handling](../03-work-queue-error-handling/)*
