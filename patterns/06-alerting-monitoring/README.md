# Pattern 6: Alerting & Monitoring

## Overview

All error handling is useless if nobody knows when things go wrong. This pattern implements **structured logging**, **error tracking via Work Queues**, and **Cloud Flow notifications** — all aligned with the Orchestrator framework's production patterns.

> See `WriteLog`, `CaptureScreenLog`, and `InsertError` subflows in [`examples/orchestrator-workqueue/`](../../examples/orchestrator-workqueue/)

## Monitoring Architecture

```
┌──────────────────────────────────────────────────────────┐
│              DESKTOP FLOW (Orchestrator)                   │
│                                                           │
│  WriteLog ──→ Log File (.txt structured)                  │
│  CaptureScreenLog ──→ Screenshots (.jpg per step)         │
│  CaptureError ──→ Error screenshot + JSON body            │
│  InsertError ──→ WQ-Errors (centralized error queue)      │
│  StartExecution / EndExecution ──→ WQ-Execution           │
│                                                           │
└───────────────────────┬──────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────┐
│                    DATA STORES                            │
│                                                           │
│  Log Files:                                               │
│     {Solution}\{CloudFlow}\Log\yyyy\MM\dd\               │
│     ├── Log-{Hostname}-{Agent}.txt                       │
│     └── Screenshots\                                     │
│         ├── 2026-02-16_143025-Error-DF0001-GenerateGuide.jpg│
│         └── ExecutionPrints\                              │
│             └── Exec_20260216_14-30-25-PrintExecDF0001.jpg│
│                                                           │
│  Work Queues:                                             │
│     ├── WQ-Execution (execution tracking)                │
│     └── WQ-Errors (centralized error log)                │
│                                                           │
└───────────────────────┬──────────────────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────────────────┐
│                 CLOUD FLOW (Alerting)                     │
│                                                           │
│  ► Desktop Flow returns ErrorBody on failure              │
│  ► Cloud Flow parses JSON → routes notification           │
│  ► Teams / Email / escalation based on severity           │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

## Implementation (Production Robin Code)

### 1. Structured Log File (WriteLog)

Every action writes a structured log line with consistent format:

```
FUNCTION WriteLog GLOBAL
    /# ================================================
    Register execution log entry.
    ================================================#/
    # Verify log directory exists
    Text.Replace Text: LogDir TextToFind: $'''\\%LogFileName%'''
        IsRegEx: False IgnoreCase: False
        ReplaceWith: $'''%''%''' Result=> AuxLogDir
    IF (Folder.IfFolderExists.DoesNotExist Path: AuxLogDir) THEN
        Text.SplitText.SplitWithDelimiter Text: AuxLogDir
            CustomDelimiter: $'''\\''' IsRegEx: False Result=> AuxLogDirParts
        Folder.Create FolderPath: $'''%AuxLogDirParts[0]%\\''' FolderName: AuxLogDir
    END

    # Capture timestamp
    DateTime.GetCurrentDateTime.Local CurrentDateTime=> Timestamp
    Text.ConvertDateTimeToText.FromCustomDateTime DateTime: Timestamp
        CustomFormat: $'''dd/MM/yyyy HH:mm:ss''' Result=> Timestamp

    # Write structured log entry
    SET LogHeaders TO $'''%in_IdExec% || %Timestamp% || %Hostname% || %AgentName%
        || %DesktopFlowName%'''
    File.WriteText File: LogDir
        TextToWrite: $'''%LogHeaders% || %SubFlowName% || %TaskLine%
        || %LogText%'''
        AppendNewLine: True
        IfFileExists: File.IfFileExists.Append
        Encoding: File.FileEncoding.Unicode
    ON ERROR
    END
END FUNCTION
```

**Log format (pipe-delimited, 8 fields):**
```
IdExec || DateTime || Hostname || Agent || DesktopFlow || Subflow || ActionLine || Message
```

**Real log output:**
```
20260216143022 || 16/02/2026 14:30:22 || SRV-RPA-01 || svc_rpa || DF0001-Orchestrator || Main || 12 || Execution started
20260216143022 || 16/02/2026 14:30:23 || SRV-RPA-01 || svc_rpa || DF0001-Orchestrator || Main || 30 || Checking process requirements
20260216143022 || 16/02/2026 14:30:24 || SRV-RPA-01 || svc_rpa || DF0001-Orchestrator || Main || 0 || Desktop Flow started: DF0001-CaptureSharePointFiles
20260216143022 || 16/02/2026 14:31:05 || SRV-RPA-01 || svc_rpa || DF0001-Orchestrator || Main || 0 || Desktop Flow ended: DF0001-CaptureSharePointFiles - Status: Success
20260216143022 || 16/02/2026 14:31:06 || SRV-RPA-01 || svc_rpa || DF0001-Orchestrator || Main || 0 || Desktop Flow started: Dequeue-WQ-MoveSpreadsheet
20260216143022 || 16/02/2026 14:35:22 || SRV-RPA-01 || svc_rpa || DF0001-Orchestrator || CaptureError || 36 || Failure: 'DesktopFlow': DF0001-GenerateGuide, 'Subflow': Main, 'Action': 5, 'Action name': Click UI Element, 'Error Message': Element not found
20260216143022 || 16/02/2026 14:35:23 || SRV-RPA-01 || svc_rpa || DF0001-Orchestrator || UpdateQueueItemError || 5 || Queue item marked with error (automatic retry by Work Queue): Element not found
20260216143022 || 16/02/2026 14:35:24 || SRV-RPA-01 || svc_rpa || DF0001-Orchestrator || InsertError || 8 || Error registered: DF=DF0001-GenerateGuide, Subflow=Main, Action=5, Msg=Element not found
20260216143022 || 16/02/2026 14:35:25 || SRV-RPA-01 || svc_rpa || DF0001-Orchestrator || Main || 999 || Execution ended - Status: Error - Element not found
```

### 2. Screenshot Evidence

#### Error Screenshot (CaptureError)
Captured automatically on any error:

```
# Inside CaptureError — screenshot with timestamp + desktop flow name
DateTime.GetCurrentDateTime.Local CurrentDateTime=> CurrentDateTime
Text.ConvertDateTimeToText.FromCustomDateTime DateTime: CurrentDateTime
    CustomFormat: $'''yyyy-MM-dd_HHmmss''' Result=> CurrentDateTime

SET EvidenceFile TO $'''%in_DirSolutions%\\%in_SolutionName%\\%in_CloudName%\\
    Log\\%LogDate%\\Screenshots\\
    %CurrentDateTime%-Error-%ErrorDesktopFlowName%.jpg'''

Workstation.TakeScreenshot.TakeScreenshotOfPrimaryScreenAndSaveToFile
    File: EvidenceFile
    ImageFormat: System.ImageFormat.Jpg
ON ERROR REPEAT 1 TIMES WAIT 2
ON ERROR
END
```

#### Execution Screenshots (CaptureScreenLog)
Optional screenshots during normal execution for audit:

```
FUNCTION CaptureScreenLog GLOBAL
    /# ================================================
    Register screenshots during normal execution.
    ================================================#/
    IF (Folder.IfFolderExists.DoesNotExist
            Path: $'''%LogDir%\\ExecutionPrints''') THEN
        Folder.Create FolderPath: LogDir FolderName: $'''ExecutionPrints'''
    END

    DateTime.GetCurrentDateTime.Local CurrentDateTime=> TimeStamp
    Text.ConvertDateTimeToText.FromCustomDateTime DateTime: TimeStamp
        CustomFormat: $'''yyyy-MM-ddThh-mm-ss''' Result=> TimeStamp

    SET ScreenshotFile TO $'''%PrintId%_%TimeStamp%-PrintExec%DesktopFlowName%.jpg'''

    Workstation.TakeScreenshot.TakeScreenshotOfPrimaryScreenAndSaveToFile
        File: $'''%LogDir%\\ExecutionPrints\\%ScreenshotFile%'''
        ImageFormat: System.ImageFormat.Jpg
    ON ERROR
    END
END FUNCTION
```

### 3. Execution Tracking (WQ-Execution)

Every execution is registered in a tracking Work Queue with start/end times and status:

```
# At start (StartExecution)
SET ExecJson TO $'''{"IdExec": "%in_IdExec%",
    "CloudName": "%in_CloudName%",
    "Environment": "%in_Environment%",
    "Hostname": "%Hostname%",
    "Username": "%AgentName%",
    "StartTime": "%ExecDateTime%",
    "Status": "Processing"}'''
WorkQueues.AddWorkQueueItem.AddWorkQueueItem
    QueueName: $'''WQ-Execution'''
    Name: $'''Exec-%in_IdExec%'''
    InputValue: ExecJson
    WorkQueueItem=> ExecWorkQueueItem

# At end (EndExecution)
SET ResultJson TO $'''{"Status": "%Status%",
    "Observation": "%Observation%",
    "EndTime": "%EndDateTimeText%"}'''
WorkQueues.UpdateWorkQueueItem.UpdateWorkQueueItem
    WorkQueueItem: ExecWorkQueueItem
    Status: WorkQueues.WorkQueueItemStatus.Processed
    ProcessingResult: ResultJson
    ProcessingNotes: $'''%Status% - %Observation%'''
```

### 4. Cloud Flow Error Response

When the Desktop Flow exits with an error, the Cloud Flow receives the error body and can route notifications:

```
# In Desktop Flow Main — exit with error body
IF Status = $'''Error''' THEN
    EXIT Code: 0 ErrorMessage: ErrorBody
END
```

**Cloud Flow handling:**
```
TRIGGER: When Desktop Flow completes

IF DesktopFlowRunStatus = "Failed" THEN
    # Parse the error body JSON
    SET ErrorInfo TO JSON.Parse(DesktopFlow.ErrorMessage)

    # Route notification based on Desktop Flow
    POST ADAPTIVE CARD TO TEAMS "#rpa-monitoring"
        Title: "RPA Error — %ErrorInfo.DesktopFlow%"
        Facts:
            - Execution: %ErrorInfo.ExecutionId%
            - Desktop Flow: %ErrorInfo.DesktopFlow%
            - Subflow: %ErrorInfo.Subflow%
            - Action: %ErrorInfo['Action name']%
            - Error: %ErrorInfo['Error Message']%
            - Screenshot: %ErrorInfo['Error Evidence']%
        Actions:
            - "View Work Queue" → make.powerautomate.com/monitor/workqueues
            - "View Log File" → file share link
END IF
```

### 5. How to Log in Your Business Logic

Pattern for adding log entries throughout your flows. Before each significant action, set the 3 logging variables and call `WriteLog`:

```
# Pattern: Log before action
SET SubFlowName TO $'''Main'''           # Which subflow
SET TaskLine TO 45                       # Action line number (for reference)
SET LogText TO $'''Processing item: %WorkQueueItem.Id%'''  # What's happening
CALL WriteLog

# ... execute the action ...

# Pattern: Log after action
SET SubFlowName TO $'''Main'''
SET TaskLine TO 48
SET LogText TO $'''Item processed successfully'''
CALL WriteLog
```

## Log File Directory Structure

```
{in_DirSolutions}\
└── {in_SolutionName}\
    └── {in_CloudName}\
        ├── Execution\
        │   └── {in_IdExec}\           ← Execution artifacts
        └── Log\
            └── yyyy\MM\dd\
                ├── Log-{Hostname}-{Agent}.txt    ← Structured log
                └── Screenshots\
                    ├── 2026-02-16_143025-Error-DF0001-GenerateGuide.jpg
                    └── ExecutionPrints\
                        └── Exec_20260216_14-30-25-PrintExec.jpg
```

## Variables for Log System

| Variable | Set in | Purpose |
|----------|--------|---------|
| `LogDir` | `Initialize` | Full path to log file |
| `LogFileName` | `Initialize` | Log file name: `Log-{Hostname}-{Agent}.txt` |
| `Hostname` | `Initialize` | Machine name (via `hostname` command) |
| `AgentName` | `Initialize` | Windows username (via `%USERNAME%`) |
| `LogDate` | `Initialize` | Date for log folder: `yyyy\\MM\\dd` |
| `SubFlowName` | Caller | Which subflow is logging |
| `TaskLine` | Caller | Action line number for reference |
| `LogText` | Caller | Log message text |
| `in_IdExec` | `StartExecution` | Execution ID (timestamp-based) |

## Initialization

All monitoring variables are set up in the `Initialize` subflow:

```
FUNCTION Initialize GLOBAL
    SET ExecFolder TO $'''%in_DirSolutions%\\%in_SolutionName%\\
        %in_CloudName%\\Execution\\'''
    SET Status TO $'''Success'''
    SET Observation TO $'''Flow executed successfully'''

    # Capture hostname
    Scripting.RunDOSCommand.RunDOSCommand
        DOSCommandOrApplication: $'''hostname'''
        StandardOutput=> Hostname
    Text.Replace Text: Hostname TextToFind: $'''(\\n|\\r|\\t)'''
        IsRegEx: True ReplaceWith: $'''%''%''' Result=> Hostname

    # Capture username
    System.GetEnvironmentVariable.GetEnvironmentVariable
        Name: $'''USERNAME''' Value=> AgentName

    # Set up log path
    DateTime.GetCurrentDateTime.Local CurrentDateTime=> LogDate
    Text.ConvertDateTimeToText.FromCustomDateTime DateTime: LogDate
        CustomFormat: $'''yyyy\\\\MM\\\\dd''' Result=> LogDate
    SET LogFileName TO $'''Log-%Hostname%-%AgentName%.txt'''
    SET LogDir TO $'''%in_DirSolutions%\\%in_SolutionName%\\
        %in_CloudName%\\Log\\%LogDate%\\%LogFileName%'''
END FUNCTION
```

## Monitoring Checklist

### Per Desktop Flow
- [ ] `BLOCK ON BLOCK ERROR all` with `CaptureError` + `InsertError`
- [ ] `Initialize` sets up log path, hostname, agent
- [ ] `WriteLog` called before/after every significant action
- [ ] `CaptureScreenLog` for optional execution screenshots
- [ ] `StartExecution` / `EndExecution` for WQ-Execution tracking
- [ ] Error screenshot captured with timestamp + DF name
- [ ] `EXIT Code: 0 ErrorMessage: ErrorBody` returns error to Cloud Flow

### Per Cloud Flow
- [ ] Parse `ErrorMessage` JSON on Desktop Flow failure
- [ ] Route Teams notification with error details
- [ ] Include link to Work Queue monitor page
- [ ] Daily summary of WQ-Execution items (optional)

---

*This is the final pattern in the framework. For the complete overview, see the [main README](../../README.md).*
