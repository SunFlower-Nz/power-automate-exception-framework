# Pattern 4: Dead Letter Queue

## Overview

A **Dead Letter Queue (DLQ)** is where items go after exhausting all retry attempts. In the Orchestrator framework, this is implemented with two mechanisms: **WQ-Errors** (centralized error log) and the Work Queue's native **Failed** status (items that exceeded `Max Retries`).

> See `InsertError` and `UpdateQueueItemError` subflows in [`examples/orchestrator-workqueue/`](../../examples/orchestrator-workqueue/)

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 Processing Queue                         │
│       (WQ-MoveSpreadsheet, WQ-GenerateGuide, etc.)       │
│                                                          │
│  Item dequeued → Processing → Processed                  │
│                       │                                  │
│                       ▼ Error                            │
│              GenericException                            │
│                       │                                  │
│          ┌────────────▼────────────┐                     │
│          │  Retries < Max Retries? │                     │
│          │  ├─ YES → Requeued      │                     │
│          │  └─ NO  → Failed        │ ← DEAD LETTER      │
│          └─────────────────────────┘                     │
└─────────────────────────────────────────────────────────┘
                       │
                       │ On any error
                       ▼
┌─────────────────────────────────────────────────────────┐
│                     WQ-Errors                            │
│            (Centralized Error Log)                       │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Name: Error-20260216143022-DF0001-GenerateGuide    │  │
│  │ Input: {                                           │  │
│  │   "ExecutionId": "20260216143022",                  │  │
│  │   "DesktopFlow": "DF0001-GenerateGuide",            │  │
│  │   "Subflow": "Main",                              │  │
│  │   "Action": "5",                                   │  │
│  │   "Action name": "Click UI Element",               │  │
│  │   "Error Message": "Element not found on screen",  │  │
│  │   "Error Evidence": "C:\\...\\screenshot.jpg"       │  │
│  │ }                                                  │  │
│  │ Status: Processed (log entry, not to be retried)   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Monitor → Work Queues → WQ-Errors → View all errors    │
└─────────────────────────────────────────────────────────┘
```

## Implementation (Production Robin Code)

### Subflow: UpdateQueueItemError (Mark Item as Failed)

When a child flow fails, the processing queue item is marked with `GenericException`. The Work Queue platform decides whether to retry or move to Failed (dead letter).

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

### Subflow: InsertError (Register in WQ-Errors)

Every error is also registered in a separate `WQ-Errors` queue for centralized tracking. This item is immediately marked as `Processed` because it's a log entry, not something to retry.

```
FUNCTION InsertError GLOBAL
    /# ================================================
    Register detailed error in the Error Work Queue and log.
    ================================================#/
    IF DesktopFlowFailed = True THEN
        # Register error in WQ-Errors
        WorkQueues.AddWorkQueueItem.AddWorkQueueItem
            QueueName: $'''WQ-Errors'''
            Priority: WorkQueues.WorkQueueItemPriority.High
            Name: $'''Error-%in_IdExec%-%ErrorDesktopFlowName%'''
            InputValue: ErrorBody
            WorkQueueItem=> ErrorWorkQueueItem

        # Mark as processed immediately (it's just a log entry)
        WorkQueues.UpdateWorkQueueItem.UpdateWorkQueueItem
            WorkQueueItem: ErrorWorkQueueItem
            Status: WorkQueues.WorkQueueItemStatus.Processed
            ProcessingNotes: $'''Error registered'''

        # ==== Write log ====
        SET SubFlowName TO $'''InsertError'''
        SET TaskLine TO 8
        SET LogText TO $'''Error registered: DF=%ErrorDesktopFlowName%,
            Subflow=%ErrorSubflowName%, Action=%ErrorActionIndex%,
            Msg=%ErrorMessage%, Evidence=%EvidenceFile%'''
        CALL WriteLog
    END
END FUNCTION
```

## DLQ Item Schema (ErrorBody JSON)

The error JSON registered in WQ-Errors contains all debugging information:

```json
{
    "ExecutionId": "20260216143022",
    "DesktopFlow": "DF0001-GenerateGuide",
    "Subflow": "Main",
    "Action": "5",
    "Action name": "Click UI Element in Window",
    "Error Message": "Element not found on screen within timeout",
    "Error Evidence": "C:\\Solutions\\PA0001\\CloudFlow\\Log\\2026\\02\\16\\Screenshots\\2026-02-16_143025-Error-DF0001-GenerateGuide.jpg"
}
```

## Error Detection Points

The error body is built in `CaptureError` with different parsing depending on error source:

| Error Source | Detection | `DesktopFlowFailed` |
|-------------|-----------|----------------------|
| Child Desktop Flow crash | `LastError.Message = 'Failed to run flow'` | `True` |
| Orchestrator action fail | Any other `LastError.Message` | `False` |

```
ERROR => LastError

IF (LastError.Message = 'Failed to run flow') = True THEN
    # Child flow crashed — parse ErrorDetails (multi-line)
    SET DesktopFlowFailed TO True
    Text.SplitText.Split Text: LastError.ErrorDetails
        StandardDelimiter: Text.StandardDelimiter.NewLine ...
ELSE
    # Orchestrator error — parse Location + Message
    SET DesktopFlowFailed TO False
    Text.SplitText.SplitWithDelimiter
        Text: $'''%LastError.Location%,%LastError.Message%''' ...
END
```

## Monitoring Dead Letter Items

### In Power Automate Portal

Navigate to **Monitor → Work queues** to see:

| Queue | Filter | What You See |
|-------|--------|--------------|
| `WQ-MoveSpreadsheet` | Status = Failed | Dead letter items (exceeded max retries) |
| `WQ-GenerateGuide` | Status = Failed | Dead letter items |
| `WQ-Errors` | All items | Centralized error log with full details |

### Cloud Flow Monitor (Optional)

```
TRIGGER: Schedule (every 30 minutes)

# Check for dead letter items across all queues
GET WORK QUEUE ITEMS FROM "WQ-MoveSpreadsheet"
    WHERE Status = "Failed"

GET WORK QUEUE ITEMS FROM "WQ-GenerateGuide"
    WHERE Status = "Failed"

IF TotalFailedCount > 0 THEN
    POST TO TEAMS "#rpa-monitoring"
        "Dead Letter Items Found
         WQ-MoveSpreadsheet: %FailedMoveCount% failed
         WQ-GenerateGuide: %FailedGuideCount% failed
         
         Check: Power Automate → Monitor → Work queues"
END IF
```

## Resolution Workflow

When an item reaches Dead Letter (Failed) status:

1. **Investigate** — Check `WQ-Errors` for the error details + screenshot path
2. **Fix root cause** — Data issue? System down? UI changed?
3. **Requeue** — Manually add item back to the processing queue with corrected data
4. **Verify** — Monitor next execution to confirm the fix

```
# Manual requeue (in a separate utility flow)
WorkQueues.AddWorkQueueItem.AddWorkQueueItem
    QueueName: $'''WQ-GenerateGuide'''
    Priority: WorkQueues.WorkQueueItemPriority.High
    Name: $'''Requeue-%OriginalName%'''
    InputValue: CorrectedItemData
    WorkQueueItem=> RequeuedItem
```

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| WQ-Errors uses `Priority: High` | Errors should be visible immediately |
| WQ-Errors items marked `Processed` immediately | They're log entries, not items to process |
| Screenshot stored as file path, not embedded | Keep WQ item size small, screenshot accessible on the machine |
| Error JSON includes all 5 fields | Desktop flow, subflow, action index, action name, message — everything needed to debug |
| Separate WQ-Errors from processing queues | Errors don't compete with real items for dequeue |

## Resolution Workflows

| Action | When | Who |
|--------|------|-----|
| **Auto-fix** | Known patterns (e.g., truncate long field) | Bot |
| **Manual fix** | Data correction needed | Data team |
| **Requeue** | After fixing root cause | RPA team |
| **Discard** | Item is obsolete or duplicate | Operations |

---

*Next: [Pattern 5 — Circuit Breaker](../05-circuit-breaker/)*
