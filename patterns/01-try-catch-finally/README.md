# Pattern 1: Try-Catch-Finally

## Overview

The foundational pattern for all Power Automate Desktop error handling. Every production flow should implement this as a minimum. This pattern uses PAD's `BLOCK ... ON BLOCK ERROR` structure to capture errors, log details, take screenshots, and clean up resources.

> See the full production implementation in [`examples/orchestrator-workqueue/`](../../examples/orchestrator-workqueue/)

## Architecture

```
┌──────────────────────────────────────────────────────┐
│                     MAIN FLOW                         │
├──────────────────────────────────────────────────────┤
│                                                       │
│  BLOCK ErrorHandler                                   │
│  ON BLOCK ERROR all                                   │
│  ├─ CALL CaptureError          ← Capture error info   │
│  ├─ CALL UpdateQueueItemError  ← Mark WQ error        │
│  ├─ CALL EndDesktopFlow        ← Log end of step      │
│  └─ CALL InsertError           ← Register in WQ-Errors│
│  END                                                  │
│                                                       │
│  ┌────────────────────────────────────────────────┐   │
│  │ TRY (Business Logic inside BLOCK)              │   │
│  │  ► CALL Initialize                             │   │
│  │  ► CALL TerminateProcesses                     │   │
│  │  ► CALL LoadConfig                             │   │
│  │  ► CALL ReadConfig                             │   │
│  │  ► CALL CaptureCredentials                     │   │
│  │  ► CALL StartExecution                         │   │
│  │  ► ... Business phases ...                     │   │
│  └────────────────────────────────────────────────┘   │
│                                                       │
│  END (BLOCK)                                          │
│                                                       │
│  ┌────────────────────────────────────────────────┐   │
│  │ FINALLY (runs always — after BLOCK)            │   │
│  │  ► CALL EndExecution                           │   │
│  │  ► CALL TerminateProcesses                     │   │
│  │  ► IF Status = 'Error' → EXIT with error       │   │
│  └────────────────────────────────────────────────┘   │
│                                                       │
└──────────────────────────────────────────────────────┘
```

## Implementation in Power Automate Desktop (Robin)

### Main Flow Structure

This is real PAD Robin code used in production. The `BLOCK ... ON BLOCK ERROR` acts as the Try-Catch, and the code after `END` (outside the block) acts as the Finally.

```
SET UpdateQueueError TO 0

BLOCK ErrorHandler
ON BLOCK ERROR all
    CALL CaptureError
    CALL UpdateQueueItemError
    CALL EndDesktopFlow
    CALL InsertError
END

    # ═══════ INITIALIZATION (inside the BLOCK = TRY) ═══════
    CALL Initialize
    CALL TerminateProcesses
    CALL LoadConfig
    CALL ReadConfig
    CALL CaptureCredentials
    CALL StartExecution

    # ==== Write log ====
    SET SubFlowName TO $'''Main'''
    SET TaskLine TO 12
    SET LogText TO $'''Execution started'''
    CALL WriteLog
    # ==== Write log ====

    # ═══════ BUSINESS LOGIC PHASES ═══════
    # ... your processing phases here ...

END

# ═══════ FINALLY (runs always — after BLOCK ends) ═══════
CALL EndExecution
CALL TerminateProcesses

IF Status = $'''Error''' THEN
    EXIT Code: 0 ErrorMessage: ErrorBody
END
```

### Subflow: CaptureError

Captures the full error context: desktop flow name, subflow, action index, action name, error message, and takes a screenshot.

```
FUNCTION CaptureError GLOBAL
    ERROR => LastError

    IF (LastError.Message = 'Failed to run flow') = True THEN
        SET DesktopFlowFailed TO True
        SET ErrorDesktopFlowName TO CurrentDF
        # Parse child flow error details
        Text.SplitText.Split Text: LastError.ErrorDetails
            StandardDelimiter: Text.StandardDelimiter.NewLine
            DelimiterTimes: 1 Result=> ErrorDetails
    ELSE
        SET DesktopFlowFailed TO False
        SET ErrorDesktopFlowName TO DesktopFlowName
        # Parse orchestrator error details
        Text.SplitText.SplitWithDelimiter
            Text: $'''%LastError.Location%,%LastError.Message%'''
            CustomDelimiter: $''',''' IsRegEx: False Result=> ErrorDetails
    END

    # Extract error components
    SET ErrorSubflowName TO ErrorDetails[0]        # Subflow name
    SET ErrorActionIndex TO ErrorDetails[1]        # Action index
    SET ErrorActionName  TO ErrorDetails[2]        # Action name
    SET ErrorMessage     TO ErrorDetails[3]        # Error message

    # Clean extracted values (remove labels like "Subflow:", "Action:", etc.)
    Text.Replace Text: ErrorSubflowName TextToFind: $'''Subflow:'''
        IsRegEx: False IgnoreCase: True ReplaceWith: $'''%''%'''
        Result=> ErrorSubflowName
    Text.Trim Text: ErrorSubflowName TrimmedText=> ErrorSubflowName

    # Log the error
    SET SubFlowName TO $'''CaptureError'''
    SET TaskLine TO 36
    SET LogText TO $'''Failure: 'DesktopFlow': %ErrorDesktopFlowName%,
        'Subflow': %ErrorSubflowName%, 'Action': %ErrorActionIndex%,
        'Action name': %ErrorActionName%, 'Error Message': %ErrorMessage%'''
    CALL WriteLog

    # ═══════ SCREENSHOT (evidence) ═══════
    DateTime.GetCurrentDateTime.Local CurrentDateTime=> CurrentDateTime
    Text.ConvertDateTimeToText.FromCustomDateTime DateTime: CurrentDateTime
        CustomFormat: $'''yyyy-MM-dd_HHmmss''' Result=> CurrentDateTime
    SET EvidenceFile TO $'''%LogDir%\\Screenshots\\
        %CurrentDateTime%-Error-%ErrorDesktopFlowName%.jpg'''
    Workstation.TakeScreenshot.TakeScreenshotOfPrimaryScreenAndSaveToFile
        File: EvidenceFile ImageFormat: System.ImageFormat.Jpg
    ON ERROR REPEAT 1 TIMES WAIT 2
    ON ERROR
    END

    # Build error JSON for Cloud Flow response
    SET ErrorBody TO { 'ExecutionId': in_IdExec,
        'DesktopFlow': ErrorDesktopFlowName,
        'Subflow': ErrorSubflowName,
        'Action': ErrorActionIndex,
        'Action name': ErrorActionName,
        'Error Message': ErrorMessageJSON,
        'Error Evidence': EvidenceFile }
    Variables.ConvertCustomObjectToJson CustomObject: ErrorBody Json=> ErrorBody

    # Set global status
    SET Status TO $'''Error'''
    SET Observation TO ErrorMessage
END FUNCTION
```

### Subflow: WriteLog (Structured Logging)

Writes structured logs with execution ID, timestamp, hostname, agent, desktop flow name, subflow, action line, and message.

```
FUNCTION WriteLog GLOBAL
    # Create log directory if needed
    IF (Folder.IfFolderExists.DoesNotExist Path: AuxLogDir) THEN
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
END FUNCTION
```

**Log output format:**
```
20260216143022 || 16/02/2026 14:30:22 || RPA-BOT-01 || svc_rpa || DF0001-Orchestrator || Main || 12 || Execution started
20260216143022 || 16/02/2026 14:30:25 || RPA-BOT-01 || svc_rpa || DF0001-Orchestrator || CaptureError || 36 || Failure: 'DesktopFlow': DF0001-GenerateGuide, ...
```

### Subflow: TerminateProcesses (Resource Cleanup)

Closes all applications opened by the RPA to prevent resource leaks, even if the flow fails.

```
FUNCTION TerminateProcesses GLOBAL
    Scripting.RunPowershellScript.RunPowershellScript Script: $'''
        $processes = Get-Process -Name msedge -ErrorAction SilentlyContinue
        foreach ($process in $processes) {
            Stop-Process -Id $process.Id -Force
        }
    ''' ScriptError=> ScriptError

    Scripting.RunPowershellScript.RunPowershellScript Script: $'''
        $processes = Get-Process -Name EXCEL -ErrorAction SilentlyContinue
        foreach ($process in $processes) {
            Stop-Process -Id $process.Id -Force
        }
    ''' ScriptError=> ScriptError
END FUNCTION
```

## Variables Reference

| Variable | Type | Purpose |
|----------|------|---------|
| `Status` | Text | Global status: `Success` or `Error` |
| `Observation` | Text | Status description |
| `ErrorBody` | Text (JSON) | Structured error payload for Cloud Flow |
| `ErrorDesktopFlowName` | Text | Which desktop flow failed |
| `ErrorSubflowName` | Text | Which subflow failed |
| `ErrorActionIndex` | Text | Action number that failed |
| `ErrorActionName` | Text | Action name that failed |
| `ErrorMessage` | Text | Error description |
| `EvidenceFile` | Text | Screenshot file path |
| `DesktopFlowFailed` | Boolean | Whether a child flow failed |
| `UpdateQueueError` | Number | Flag: 1 = update WQ item on error |

## Key Concepts

### BLOCK vs ON ERROR

| Feature | `BLOCK ... ON BLOCK ERROR` | `ON ERROR` (per action) |
|---------|---------------------------|-------------------------|
| **Scope** | Wraps multiple actions | Single action only |
| **Use case** | Global error handler for entire flow | Action-level retry/skip |
| **In Orchestrator** | Main error handler | `DequeueWorkQueueItem` (empty queue = exit loop) |

### Error Information Available

| Property | Description | Example |
|----------|-------------|---------|
| `LastError.Message` | Error message | `"Failed to run flow"` |
| `LastError.Location` | Where it occurred | `"Subflow: Main, Action: 5"` |
| `LastError.ErrorDetails` | Full details (child flows) | Multi-line error breakdown |

## Best Practices

### DO
- Use `BLOCK ON BLOCK ERROR all` as the outermost error handler
- Capture `ERROR => LastError` as the first action in the catch
- Always take a screenshot — the single most useful debugging artifact
- Build a structured JSON error body for the Cloud Flow to consume
- Call `TerminateProcesses` in **both** the error handler and after the BLOCK (finally)
- Use structured log format with `||` delimiter for easy parsing
- Include execution ID (`in_IdExec`) in every log line for traceability

### DON'T
- Don't catch errors silently (always log + screenshot)
- Don't leave browser/Excel processes running after failure
- Don't use generic error messages — parse the full error context
- Don't skip the cleanup after the BLOCK (resource leaks break next execution)
- Don't store sensitive data (passwords) in error logs

## Anti-Patterns

### Silent Failure
```
BLOCK ErrorHandler
ON BLOCK ERROR all
    # Nothing here — error is swallowed silently
END
    # Business logic...
END
```

### No Resource Cleanup
```
BLOCK ErrorHandler
ON BLOCK ERROR all
    CALL CaptureError
    CALL InsertError
    # Missing: TerminateProcesses — next execution finds browser/Excel still open
END
    CALL TerminateProcesses   # Only runs on success path
    # Business logic...
END
```

### No Screenshot
```
BLOCK ErrorHandler
ON BLOCK ERROR all
    CALL CaptureError       # But CaptureError skips the screenshot — no evidence
    CALL InsertError
END
```

---

*Next: [Pattern 2 — Retry with Exponential Backoff](../02-retry-exponential-backoff/)*
