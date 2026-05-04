# Risk level 3: Structured logging

**AI System Privacy Audit: Documentation Quality Compliance Checker**

!!! note Priority 3

    Structured logging for coding workflow tracking

## 1. System Diagram

![](images/raw/compliance-checker-2026-05-02.png)

## 2. Data Flow Analysis

| Data Flow    | Source  | Destination | Encrypted? | Logged? | Priority |
| ------------ | ------- | ----------- | ---------- | ------- | -------- |
| User input   | Web app | LLM API     | In-transit | Yes     | High     |
| Model output | LLM API | Database    | No         | 50%     | Medium   |

## 3. Sensitive Data

### Sensitive Data Name: Log entries

- **Category:**
- **Examples:**
- **Why Sensitive:**
- **Current Protection:**
- **Risk (or Harm) if Exposed:**

## 4. Privacy Risks

### Risk 3: Structured logging for coding workflow tracking

- **Priority:** [Medium]
- **Risk Category:** [model/input/additional data/other (please describe!)]
- **Potential Harm/Impact:** What happens if this risk isn't addressed?
- **Ability to Implement Control:** [Low/Medium/High]
- **Recommended controls:**
  - (your first guess, ok to not know and leave blank)

---

## Additional information from the repo

Brief summary of how this repository uses **structlog** for JSON-friendly, field-oriented logs, with emphasis on **document review and orchestration workflows** (not IDE “coding” sessions).

### Stack and configuration

- **Library:** [structlog](https://www.structlog.org/) (see `pyproject.toml` for the pinned version).
- **Central setup:** `src/doc_quality/core/logging_config.py` — `configure_logging(log_level, log_format)` merges context variables, adds log level and ISO timestamps, then renders either **JSON** (`LOG_FORMAT=json`) or a **console** dev renderer (`LOG_FORMAT=console`, default).
- **Bootstrap:** the main FastAPI app calls `configure_logging` in its lifespan hook (`src/doc_quality/api/main.py`). The standalone orchestrator (`services/orchestrator/`) uses `structlog.get_logger(__name__)` for workflow services; it does not currently reuse `logging_config.configure_logging` in its lifespan (startup still emits structured `orchestrator_starting` / `orchestrator_stopping` events).

For full observability (OTel, metrics, audit tables), see `OBSERVABILITY_LOGGING_README.md` at the repo root.

### Request-scoped context (correlating HTTP to downstream work)

HTTP middleware in `src/doc_quality/api/main.py` clears and binds **structlog context variables** per request:

- `request_id` — from `X-Request-ID` or a new UUID.
- `correlation_id` — from `X-Correlation-ID` or the same as `request_id`.
- `trace_id` — OpenTelemetry trace hex when tracing is active, then merged into context before the access log line.

Each request logs a single structured event, `http_request`, with `method`, `path`, `status`, `duration_ms`, and `trace_id`. Those context vars are merged into subsequent log lines from the same request when code uses the shared processor chain (see `merge_contextvars` in `logging_config.py`).

### Workflow tracking (orchestrator)

The **document review flow** (`services/orchestrator/src/doc_quality_orchestrator/flows/document_review_flow.py`) logs orchestration decisions and outcomes with stable **event names** and IDs:

| Event | Purpose |
| --- | --- |
| `crewai_not_available_forced_fallback` / `crewai_kill_switch_active` | Routing warnings when CrewAI path is unavailable or disabled |
| `workflow_run_timeout` | Global run timeout with `run_id` and limit |
| `crew_workflow_starting` / `crew_workflow_completed` | Crew path: `run_id`, `trace_id`, `workflow_id`, optional `document_id`, verifier and validator summary fields |
| `scaffold_workflow_completed` | Single-agent fallback path completion with `run_id`, `trace_id`, `workflow_id`, `provider` |

**Dual trail:** the flow also posts structured payloads to the backend **Skills audit** API (`skills_api.log_event`) for events such as `workflow_routing_decision`, `crew_workflow_completed`, and `workflow_run_completed`. Those rows support compliance-style audit; structlog lines support live operations and log aggregation.

### Step-level execution (limits and crew bookkeeping)

`services/orchestrator/src/doc_quality_orchestrator/runtime_limits.py` implements `RuntimeLimitEnforcer`, which logs:

- `step_limit_exceeded` / `token_limit_exceeded` — limit breaches with `reason` and `run_id`
- `step_execution_recorded` — each recorded step with `step_id`, `agent_id`, `step_number`, `attempt`, `status`, `duration_seconds`, `tokens_consumed`

Together with flow-level events, this gives a **granular timeline** of agent/tool steps inside a workflow run.

### Backend domain workflows (Skills API and HITL)

Structured logs in `src/doc_quality/services/skills_service.py` track the document pipeline: e.g. `skill_document_persisted`, `skill_document_workflow_status_updated`, `skill_text_extracted`, `skill_finding_written`, and `skill_event_logged` (mirrors persisted audit events).

Human-in-the-loop state changes are logged in `src/doc_quality/services/hitl_workflow.py` (`review_created`, `review_not_found`, `review_status_updated`). Document analysis and compliance paths use similarly named events in `document_analyzer.py`, `compliance_checker.py`, etc.

### Conventions for operators

1. Prefer **keyword arguments** on log calls (`logger.info("event_name", key=value)`) so JSON output has queryable fields.
2. Correlate cross-service behavior with **`run_id` / `trace_id`** from orchestrator logs and **`request_id` / `correlation_id`** from API middleware.
3. Use **`LOG_FORMAT=json`** in production so log platforms can index `event`, `timestamp`, and custom fields without regex parsing.

### Source pointers

| Area | Path |
| --- | --- |
| Logging setup | `src/doc_quality/core/logging_config.py` |
| HTTP + contextvars | `src/doc_quality/api/main.py` |
| Flow + crew/scaffold | `services/orchestrator/.../flows/document_review_flow.py` |
| Step limits | `services/orchestrator/.../runtime_limits.py` |
| Skills + audit mirror | `src/doc_quality/services/skills_service.py` |
| HITL | `src/doc_quality/services/hitl_workflow.py` |
