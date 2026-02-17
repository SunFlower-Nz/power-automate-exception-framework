# Power Automate Exception Framework

Production-grade error handling patterns for Power Automate Desktop and Cloud flows. Battle-tested in enterprise RPA implementations processing 1,000+ items/day with 99.7% success rate.

## Why This Framework?

Most Power Automate flows fail in production because they lack proper error handling. This framework provides **6 proven patterns** with real PAD Robin code, backed by a **production Orchestrator** example you can download, analyze, and adapt.

## Patterns

| # | Pattern | Use Case | Complexity |
|---|---------|----------|------------|
| 1 | [Try-Catch-Finally](patterns/01-try-catch-finally/) | `BLOCK ON BLOCK ERROR` + error capture + screenshot + cleanup | Low |
| 2 | [Retry with Exponential Backoff](patterns/02-retry-exponential-backoff/) | Work Queue `GenericException` auto-retry + manual retry for APIs | Medium |
| 3 | [Work Queue Error Handling](patterns/03-work-queue-error-handling/) | Item-level isolation with `DequeueWorkQueueItem` loops | Medium |
| 4 | [Dead Letter Queue](patterns/04-dead-letter-queue/) | `WQ-Errors` centralized error tracking + `Failed` items | High |
| 5 | [Circuit Breaker](patterns/05-circuit-breaker/) | Pre-condition checks + state-based failure prevention | High |
| 6 | [Alerting & Monitoring](patterns/06-alerting-monitoring/) | Structured logging + WQ tracking + Cloud Flow notifications | Medium |

## Production Example: Orchestrator

The [`examples/orchestrator-workqueue/`](examples/orchestrator-workqueue/) folder contains a **real production Orchestrator** — a PAD orchestration framework that implements all 6 patterns. It was migrated from SQL Server to Work Queues, eliminating database dependency entirely.

### Architecture

```
Cloud Flow (trigger)
    │
    ▼
Orchestrator (Desktop Flow)
    ├── Initialize              ← Variables, hostname, log path
    ├── TerminateProcesses      ← Kill browser/Excel (cleanup)
    ├── LoadConfig              ← JSON config from Cloud Flow
    ├── ReadConfig              ← Read config values
    ├── CaptureCredentials      ← Credentials from JSON
    ├── StartExecution          ← Register in WQ-Execution
    │
    ├── PHASE 1: Populate Work Queues
    ├── PHASE 2: LOOP WQ-MoveSpreadsheet (Dequeue → Process → Processed)
    ├── PHASE 3: LOOP WQ-GenerateGuide (Login check → Dequeue → Process)
    ├── PHASE 4: LOOP WQ-MoveSpreadsheet (2nd pass)
    ├── PHASE 5: LOOP WQ-GenerateReport
    │
    ├── EndExecution            ← Update WQ-Execution
    └── TerminateProcesses      ← Final cleanup

Error Handler (BLOCK ON BLOCK ERROR all):
    ├── CaptureError            ← Parse error + screenshot
    ├── UpdateQueueItemError    ← GenericException (auto-retry)
    ├── EndDesktopFlow          ← Log step end
    └── InsertError             ← Register in WQ-Errors
```

### Example Files

| File | Pattern | Purpose |
|------|---------|---------|
| `Main.txt` | 1, 2, 3 | Orchestrator with BLOCK error handler + WQ loops |
| `f_CapturarErro.txt` | 1 | CaptureError: parse, screenshot, JSON body |
| `f_EscreverLog.txt` | 6 | WriteLog: structured logging (pipe-delimited) |
| `f_AtualizarErroItemFila.txt` | 2, 4 | UpdateQueueItemError: mark WQ item as GenericException |
| `f_InserirErro.txt` | 4 | InsertError: register error in WQ-Errors |
| `f_CapturaTelaLog.txt` | 6 | CaptureScreenLog: execution screenshots |
| `f_InicioExecucao.txt` | 6 | StartExecution: register execution start in WQ-Execution |
| `f_FimExecucao.txt` | 6 | EndExecution: register execution end |
| `f_Init.txt` | 6 | Initialize: variables, hostname, log path |
| `f_FinalizarProcessos.txt` | 1 | TerminateProcesses: kill processes (cleanup) |
| `f_CarregarConfig.txt` | — | LoadConfig: parse JSON config |
| `f_LerConfig.txt` | — | ReadConfig: read config values |
| `f_CapturarCredenciais.txt` | — | CaptureCredentials: read credentials from JSON |
| `f_InicioDesktopflow.txt` | 6 | StartDesktopFlow: log child flow start |
| `f_FimDesktopflow.txt` | 6 | EndDesktopFlow: log child flow end |
| `GUIA_IMPLEMENTACAO.md` | — | Step-by-step migration guide (SQL → WQ) |

## Impact

These patterns achieved in production:
- **99.7% success rate** across 50+ production flows
- **85% reduction** in manual intervention
- **Zero data loss** during failures (WQ-Errors preserves every error)
- **< 5 min MTTR** (Mean Time To Recovery) via structured logs + screenshots

## Quick Start

1. Read the patterns in order (1 → 6)
2. Study the [`examples/orchestrator-workqueue/`](examples/orchestrator-workqueue/) code
3. Read the [`GUIA_IMPLEMENTACAO.md`](examples/orchestrator-workqueue/GUIA_IMPLEMENTACAO.md) for the migration guide
4. Copy the subflow `.txt` files into your PAD designer
5. Adapt the `Main.txt` phases to your business process

## Tech Stack

- **Power Automate Desktop**: Robin language (`.txt` subflow exports)
- **Power Automate Cloud**: Trigger, scheduling, notifications
- **Work Queues**: Native PAD Work Queues (no SQL Server dependency)
- **Logging**: Structured text files (pipe-delimited) + screenshots

## License

MIT
