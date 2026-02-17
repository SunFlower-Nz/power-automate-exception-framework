# Pattern 3: Work Queue Error Handling

## Overview

When processing batches of items (invoices, files, records), you need **item-level error isolation**. One failed item should NOT stop the entire batch. This pattern uses Power Automate Work Queues for enterprise-grade batch processing with automatic retry, sequential phase orchestration, and centralized error tracking.

> See the full production Orchestrator in [`examples/orchestrator-workqueue/`](../../examples/orchestrator-workqueue/) — 15 subflows + implementation guide.

## Architecture (Orchestrator Pattern)

```
┌──────────────────────────────────────────────────────────┐
│                   CLOUD FLOW (Trigger)                    │
│  ► Schedule or Manual trigger                             │
│  ► Passes in_ConfigJson, in_Environment, in_CloudName     │
│  ► Calls Desktop Flow (Orchestrator)                      │
└─────────────────────┬────────────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────────────┐
│              ORCHESTRATOR (Desktop Flow)                   │
│                                                           │
│  BLOCK ErrorHandler                                       │
│  ON BLOCK ERROR all → CaptureError → InsertError          │
│                                                           │
│  ┌─ PHASE 1: DF0001-CaptureSharePointFiles ─────────────┐ │
│  │  Runs once — populates all Work Queues                │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌─ PHASE 2: LOOP WQ-MoveSpreadsheet ──────────────────┐ │
│  │  Dequeue → Run DF0001-MoveSpreadsheet → Mark Processed│ │
│  │  ON ERROR (empty queue) → EXIT LOOP                   │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌─ PHASE 3: LOOP WQ-GenerateGuide ────────────────────┐ │
│  │  Check login → Dequeue → Run DF0001-GenerateGuide →   │ │
│  │  Mark Processed. ON ERROR → EXIT LOOP                 │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌─ PHASE 4: LOOP WQ-MoveSpreadsheet (2nd pass) ───────┐ │
│  │  Process newly generated files from Phase 3           │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                           │
│  ┌─ PHASE 5: LOOP WQ-GenerateReport ───────────────────┐ │
│  │  Dequeue → Run DF0001-GenerateReport → Mark Processed │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                           │
│  END (BLOCK)                                              │
│  EndExecution → TerminateProcesses                        │
└───────────────────────────────────────────────────────────┘

Work Queues (configured in portal):
┌──────────────────────┬─────────────┬──────────┐
│ WQ-MoveSpreadsheet   │ Max Retry: 3│ Data flow│
│ WQ-GenerateGuide     │ Max Retry: 3│ Data flow│
│ WQ-GenerateReport    │ Max Retry: 3│ Data flow│
│ WQ-Execution         │ Max Retry: 0│ Tracking │
│ WQ-Errors            │ Max Retry: 0│ Logging  │
└──────────────────────┴─────────────┴──────────┘
```

## Item States

```
Queued → Processing → Processed
              │
              ├──→ GenericException (auto-requeue if retries left)
              │         └──→ Queued (retry)
              │
              └──→ Failed (max retries exceeded)
```

## Implementation (Production Robin Code)

### 1. Main Flow — Work Queue Loop Pattern

Each phase follows the same pattern: `LOOP → Dequeue → Process → Mark Processed → END LOOP`.

```
# ================================================================
# PHASE 2: Loop WQ-MoveSpreadsheet
# ================================================================
IF (CurrentDF = 'DF0001-MoveSpreadsheet') = True THEN
    LOOP WHILE (1) = (1)
        # ─── DEQUEUE ───
        SET CurrentDF TO $'''Dequeue-WQ-MoveSpreadsheet'''
        CALL StartDesktopFlow
        WorkQueues.DequeueWorkQueueItem.DequeueWorkQueueItem
            QueueName: $'''WQ-MoveSpreadsheet'''
            WorkQueueItem=> WorkQueueItem
        ON ERROR
            # Empty queue = no more items → exit loop normally
            SET UpdateQueueError TO 0
            CALL EndDesktopFlow
            EXIT LOOP
        END
        CALL EndDesktopFlow

        # ─── PROCESS ───
        SET UpdateQueueError TO 1
        SET QueueItemId TO WorkQueueItem.Id
        SET CurrentDF TO $'''DF0001-MoveSpreadsheet'''
        CALL StartDesktopFlow
        @@flowname: 'DF0001-MoveSpreadsheet'
        External.RunFlow FlowId: 'e731fb3a-...'
            @In_ItemData: WorkQueueItem.Input
            @In_Environment: in_Environment
            @In_IdExec: in_IdExec
            @In_LogDir: LogDir
            @in_CloudName: in_CloudName
        CALL EndDesktopFlow

        # ─── MARK SUCCESS ───
        WorkQueues.UpdateWorkQueueItem.UpdateWorkQueueItem
            WorkQueueItem: WorkQueueItem
            Status: WorkQueues.WorkQueueItemStatus.Processed
            ProcessingNotes: $'''Processed successfully'''
    END
    SET CurrentDF TO $'''DF0001-GenerateGuide'''
END
```

### 2. Phase with Pre-Condition Check (Login Verification)

Some phases need to verify a system is ready before processing each item:

```
# ================================================================
# PHASE 3: Loop WQ-GenerateGuide
# Checks portal window before processing each item
# ================================================================
IF (CurrentDF = 'DF0001-GenerateGuide'
    OR CurrentDF = 'DF0000-LoginPortal') = True THEN
    LOOP WHILE (1) = (1)
        # ─── DEQUEUE ───
        SET CurrentDF TO $'''Dequeue-WQ-GenerateGuide'''
        CALL StartDesktopFlow
        WorkQueues.DequeueWorkQueueItem.DequeueWorkQueueItem
            QueueName: $'''WQ-GenerateGuide'''
            WorkQueueItem=> WorkQueueItem
        ON ERROR
            SET UpdateQueueError TO 0
            CALL EndDesktopFlow
            EXIT LOOP
        END
        CALL EndDesktopFlow
        SET UpdateQueueError TO 1
        SET QueueItemId TO WorkQueueItem.Id

        # ─── PRE-CONDITION: Check if portal is open ───
        IF (UIAutomation.IfWindow.IsNotOpenByTitleClass
                Title: $'''Guide Entry - Consultation*'''
                Class: $'''''') THEN
            CALL TerminateProcesses
            SET CurrentDF TO $'''DF0000-LoginPortal'''
            CALL StartDesktopFlow
            External.RunFlow FlowId: 'e053320f-...'
                @In_User: SystemUser
                @In_Password: SystemPassword
                @In_Environment: in_Environment
            CALL EndDesktopFlow
        END

        # ─── PROCESS ───
        SET CurrentDF TO $'''DF0001-GenerateGuide'''
        CALL StartDesktopFlow
        External.RunFlow FlowId: '3f434cc8-...'
            @In_ItemData: WorkQueueItem.Input
            @In_LogDir: LogDir
            @In_IdExec: in_IdExec
            @in_Environment: in_Environment
        CALL EndDesktopFlow

        # ─── MARK SUCCESS ───
        WorkQueues.UpdateWorkQueueItem.UpdateWorkQueueItem
            WorkQueueItem: WorkQueueItem
            Status: WorkQueues.WorkQueueItemStatus.Processed
            ProcessingNotes: $'''Processed successfully'''
    END
END
```

### 3. Resumable Execution

The Orchestrator supports **resuming from any phase** after a crash. The `in_CurrentDF` parameter tells it which phase to start from:

```
SET CurrentDF TO in_CurrentDF

IF in_IdExec.Length = 0 THEN
    # First execution — start from the beginning
    CALL StartExecution
    SET SubFlowName TO $'''Main'''
    SET LogText TO $'''Execution started'''
    CALL WriteLog
    SET CurrentDF TO $'''DF0001-CaptureSharePointFiles'''
ELSE
    # Resumed execution — continue from where it stopped
    SET SubFlowName TO $'''Main'''
    SET LogText TO $'''Resuming execution'''
    CALL WriteLog
    SET CurrentDF TO in_CurrentDF
END
```

### 4. Execution Tracking (WQ-Execution)

```
FUNCTION StartExecution GLOBAL
    # Generate unique execution ID from timestamp
    DateTime.GetCurrentDateTime.Local CurrentDateTime=> ExecDateTime
    Text.ConvertDateTimeToText.FromCustomDateTime DateTime: ExecDateTime
        CustomFormat: $'''yyyyMMddHHmmss''' Result=> in_IdExec

    # Register in execution tracking queue
    SET ExecJson TO $'''{"IdExec": "%in_IdExec%",
        "CloudName": "%in_CloudName%",
        "Environment": "%in_Environment%",
        "Hostname": "%Hostname%",
        "Username": "%AgentName%",
        "StartTime": "%ExecDateTime%",
        "Status": "Processing"}'''
    WorkQueues.AddWorkQueueItem.AddWorkQueueItem
        QueueName: $'''WQ-Execution'''
        Priority: WorkQueues.WorkQueueItemPriority.Normal
        Name: $'''Exec-%in_IdExec%'''
        InputValue: ExecJson
        WorkQueueItem=> ExecWorkQueueItem
END FUNCTION
```

```
FUNCTION EndExecution GLOBAL
    DateTime.GetCurrentDateTime.Local CurrentDateTime=> EndDateTime
    Text.ConvertDateTimeToText.FromCustomDateTime DateTime: EndDateTime
        CustomFormat: $'''dd/MM/yyyy HH:mm:ss''' Result=> EndDateTimeText

    SET ResultJson TO $'''{"Status": "%Status%",
        "Observation": "%Observation%",
        "EndTime": "%EndDateTimeText%"}'''
    WorkQueues.UpdateWorkQueueItem.UpdateWorkQueueItem
        WorkQueueItem: ExecWorkQueueItem
        Status: WorkQueues.WorkQueueItemStatus.Processed
        ProcessingResult: ResultJson
        ProcessingNotes: $'''%Status% - %Observation%'''

    SET SubFlowName TO $'''Main'''
    SET TaskLine TO 999
    SET LogText TO $'''Execution ended - Status: %Status% - %Observation%'''
    CALL WriteLog
END FUNCTION
```

## Error Handling Flow (on any error)

```
┌────────────────────────────────────────────────────┐
│  Error occurs in any action inside BLOCK            │
│                     │                               │
│    ┌────────────────▼────────────────────┐          │
│    │ 1. CaptureError                     │          │
│    │    ► ERROR => LastError              │          │
│    │    ► Parse: DF, Subflow, Action, Msg │          │
│    │    ► Take screenshot                 │          │
│    │    ► Build JSON error body           │          │
│    │    ► SET Status TO 'Error'           │          │
│    └────────────────┬────────────────────┘          │
│    ┌────────────────▼────────────────────┐          │
│    │ 2. UpdateQueueItemError             │          │
│    │    ► IF UpdateQueueError = 1        │          │
│    │    ► UpdateWorkQueueItem             │          │
│    │      Status: GenericException        │          │
│    │    ► WQ decides: retry or fail       │          │
│    └────────────────┬────────────────────┘          │
│    ┌────────────────▼────────────────────┐          │
│    │ 3. EndDesktopFlow                   │          │
│    │    ► Log: "Desktop Flow ended"      │          │
│    └────────────────┬────────────────────┘          │
│    ┌────────────────▼────────────────────┐          │
│    │ 4. InsertError                      │          │
│    │    ► AddWorkQueueItem to WQ-Errors  │          │
│    │    ► Mark as Processed (log only)   │          │
│    │    ► Log error details              │          │
│    └─────────────────────────────────────┘          │
└────────────────────────────────────────────────────┘
```

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| `ON ERROR → EXIT LOOP` on Dequeue | Empty queue is not an error — it's the normal end condition |
| `UpdateQueueError` flag | Prevents updating a non-existent WQ item before dequeue |
| `GenericException` status | Lets the Work Queue platform handle retry logic |
| `WQ-Errors` as separate queue | Decouples error logging from processing queues |
| `WQ-Execution` for tracking | Provides audit trail without database dependency |
| Sequential phases with `CurrentDF` | Enables resume from any phase after crash |
| `External.RunFlow` for child flows | Isolates business logic from orchestration |

## Key Benefits

1. **Item-level isolation**: One failed item doesn't stop the batch — `GenericException` requeues it
2. **Automatic retry**: Work Queue handles retry count and requeue transparently
3. **Resumable**: Crashed executions can resume from the exact phase via `in_CurrentDF`
4. **No database dependency**: Eliminated SQL Server — all state in Work Queues
5. **Audit trail**: Every execution, step, and error is logged with structured format
6. **Scalable**: Could parallelize with multiple workers on the same queue

---

*Next: [Pattern 4 — Dead Letter Queue](../04-dead-letter-queue/)*
