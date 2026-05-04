# AI System Privacy Audit: Application Telemetry and Workflow Tracing

System in scope: `doc_quality_compliance_check` — quality telemetry persistence layer (`quality_observations`), workflow audit trail (`audit_events`), OpenTelemetry request tracing, Prometheus metrics, and the observability API surface.

## 1. System Diagram

![](images/raw/compliance-checker-2026-05-02.png)

Telemetry-relevant architecture facts used in this risk sheet:

- The system implements a **multi-layer observability stack**: structlog structured logs, OpenTelemetry (OTEL) request spans, Prometheus metrics, and quality/evaluation telemetry persisted to PostgreSQL.
- **`quality_observations`** table (`QualityObservationORM`): stores AI quality signals per workflow step including a free-form `payload` JSON column that the application uses to embed raw `llm_prompt` and `llm_output` strings (confirmed in `test_observability_api.py` and `OBSERVABILITY_LOGGING_README.md`).
- **`audit_events`** table: append-only compliance trail with full provenance fields; the `payload` JSON column carries event-specific data, design intent is "no PII", but caller-controlled.
- **`agent_telemetry`** (referenced in persistence DoD): planned shorter-retention stream for non-compliance-critical operational signals — distinct from `audit_events`.
- Observability API (`/api/v1/observability/*`) is accessible to service clients (`allow_service=True`) in addition to human roles (`qm_lead`, `auditor`, `riskmanager`, `architect`).
- The frontend Admin/Observability page renders raw `llm_prompt` / `llm_output` pairs in a "Rich GenAI Trace Payload" block and provides a CSV export of these pairs to the local filesystem.
- OTEL exporter is configurable (`none` / `console` / `otlp`); if `otlp`, trace data leaves the system to an external endpoint.

## 2. Data Flow Analysis

| Data Flow | Source | Destination | Encrypted? | Logged? | Priority |
| --- | --- | --- | --- | --- | --- |
| Quality observation posted by workflow agent or orchestrator | Backend service / CrewAI orchestrator | `POST /api/v1/observability/quality-observations` → `quality_observations` (PostgreSQL) | In-transit (HTTPS/TLS) | Persisted; no explicit pre-write redaction policy documented | High |
| Raw LLM prompt + output embedded in `payload` JSON | Workflow agent (e.g., `research_agent`, `document_analyzer`) | `quality_observations.payload` column (PostgreSQL, at-rest) | At-rest DB controls | Persisted indefinitely — no TTL or purge on `quality_observations` visible in schema | High |
| LLM trace pairs returned via observability API | `quality_observations` table | `GET /api/v1/observability/llm-traces` → API response | In-transit (HTTPS/TLS) | Accessible to both human roles and service clients (`allow_service=True`) | High |
| Audit event appended by Skills API or orchestrator | Backend / orchestrator | `audit_events` table (PostgreSQL, append-only) | At-rest DB controls | Permanently retained; caller controls `payload` content; design intent is "no PII" but not enforced at DB layer | High |
| Prometheus metrics scraped from `/metrics` | Backend API | Prometheus scraper / monitoring stack | In-transit (depends on infrastructure setup) | No authentication on `/metrics` endpoint observed in code review | Medium |
| OTEL span data exported (when `TRACING_EXPORTER=otlp`) | FastAPI middleware / OTEL SDK | External OTLP collector endpoint | In-transit (TLS depends on OTLP endpoint config) | Spans include method, path, status, user-agent; path may carry document IDs or session context | High |
| OTEL span data emitted to console (when `TRACING_EXPORTER=console`) | FastAPI middleware / OTEL SDK | stdout / log aggregator | Same as structured log stream | Spans visible in application logs — see also Risk Sheet: Structured Logging | Medium |
| CSV export of prompt/output pairs | Frontend observability page | Browser local file download | In-browser object URL | Creates an uncontrolled copy outside the system; no audit log of export action | High |
| `agent_telemetry` operational signals (planned, shorter retention) | Orchestrator/agents | Dedicated table (retention config TBD) | At-rest DB controls | Retention TTL defined conceptually but not yet implemented in observed schema | Medium |
| Quality summary / workflow breakdown returned via API | `quality_observations` aggregation | `GET /api/v1/observability/quality-summary`, `/workflow-components` | In-transit (HTTPS/TLS) | Aggregated (no raw content); service-client accessible | Low |

### Corrected interpretation for privacy and GDPR

- The `quality_observations.payload` column is the highest-risk telemetry surface: it stores raw `llm_prompt` and `llm_output` strings, both of which may contain personal data from submitted documents, stakeholder names, and reviewer identifiers.
- The `audit_events.payload` column carries a similar risk: while the design intent is "no PII", this is a convention, not a technical control — any caller can write personal data into the payload.
- Exporting prompt/output pairs to a local CSV file is a GDPR incident risk: it creates an unregulated copy of potentially personal content on a user's device with no audit trail.
- GDPR Art. 5(1)(b) (purpose limitation) applies: telemetry collected for model quality improvement must not be repurposed for user profiling or commercial model training.

## 3. Sensitive Data

### Sensitive Data: Raw LLM Prompt and Output in `quality_observations.payload`

- **Category:** Content-derived personal data — GDPR Art. 4(1); may include direct identifiers
- **Examples:** `payload.llm_prompt` (document text submitted by user, stakeholder names, reviewer assignments), `payload.llm_output` (generated compliance summaries restating personal context), `payload.provider`, `payload.model_used`, `payload.llm_temperature`
- **Why Sensitive:** Prompt context is assembled from user-submitted documents which routinely contain names, emails, project identifiers, and business-sensitive content; the output may restate or summarise that content; both are persisted to a queryable table
- **Current Protection:** Role-gated observability API; PostgreSQL at-rest controls; no pre-write redaction
- **Risk (or Harm) if Exposed:** Unauthorised access to document content and personal data via telemetry query; GDPR breach; misuse for model training outside the original purpose; profiling of document submitters from trace history

### Sensitive Data: Audit Event Payload (`audit_events.payload`)

- **Category:** Compliance-critical record with caller-controlled content
- **Examples:** `payload.roles` (user role at login), `payload.remember_me` flag, event-specific context fields, any free-form data inserted by orchestrator or Skills API callers
- **Why Sensitive:** Append-only and intended for long retention; no technical control prevents a caller from writing personal data into the `payload` column; combined with `actor_id` (user email) it forms a rich personal profile
- **Current Protection:** Append-only service contract; `sanitize_text()` applied to provenance fields; role-gated `audit-trail` API
- **Risk (or Harm) if Exposed:** Compliance-critical records leaking personal data; inability to delete under GDPR Art. 17 due to append-only constraint without a selective redaction mechanism; cumulative re-identification from cross-event correlation

### Sensitive Data: OTEL Span Attributes (when exporter is configured)

- **Category:** Operational metadata potentially linked to individuals
- **Examples:** HTTP `path` attribute (e.g., `/api/v1/documents/doc-abc123` — document ID in path), `user_agent`, `http.method`, `http.status_code`, `trace_id` / `span_id` (linkable back to session)
- **Why Sensitive:** When `TRACING_EXPORTER=otlp`, span data is sent to an external collector; path values can embed document or session identifiers; user-agent can contribute to fingerprinting
- **Current Protection:** `TRACING_ENABLED` and exporter type are config-controlled; sampling ratio `1.0` by default (full capture)
- **Risk (or Harm) if Exposed:** Cross-border transfer of operational metadata to external SaaS OTEL backend without a GDPR Art. 28 Data Processor Agreement; correlation of user activity across sessions via trace linkage

### Sensitive Data: Frontend CSV Export of Prompt/Output Pairs

- **Category:** Uncontrolled copy of telemetry data including personal content
- **Examples:** Exported `observability_prompt_output_pairs_*.csv` file containing `prompt`, `output`, `source_component`, `trace_id`, timestamps
- **Why Sensitive:** Created on the user's local filesystem; outside system access controls, retention policy, and audit log; may contain personal data from documents processed during the export window
- **Current Protection:** Requires authenticated user in an authorised role to reach the export function
- **Risk (or Harm) if Exposed:** Untracked personal data copies; shadow copies on employee devices; GDPR Art. 32 security of processing gap; no mechanism to enforce deletion when user leaves the organisation

## 4. Privacy Risks

### Risk 1: Raw prompt/output content persisted in `quality_observations.payload` without redaction

- **Priority:** High
- **Risk Category:** Data minimisation and telemetry redaction
- **GDPR Reference:** Art. 5(1)(b) — purpose limitation; Art. 5(1)(c) — data minimisation; Art. 25 — privacy by design
- **Potential Harm/Impact:** Personal data from user documents (names, identifiers, document passages) is stored in a queryable telemetry table with no documented TTL; accessible to all authorised roles and service clients; can be bulk-exported to CSV
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Apply a mandatory redaction/scrubbing step in `create_quality_observation()` service before persistence: strip or hash identifiable content from `payload.llm_prompt` and `payload.llm_output` (e.g., replace with content hash + length metadata).
  - Define a schema for permitted `payload` fields and reject or sanitise unexpected keys.
  - Set an explicit retention TTL on `quality_observations` (e.g., 90 days for operational data; differentiate from `audit_events` compliance retention).
  - Separate "operational telemetry" (latency, scores, flags) from "content telemetry" (prompt/output text) into different storage tiers with different access policies.

### Risk 2: `audit_events.payload` has no technical enforcement of the "no PII" design intent

- **Priority:** High
- **Risk Category:** Audit trail data governance and GDPR compliance
- **GDPR Reference:** Art. 5(1)(c) — data minimisation; Art. 17 — right to erasure (conflict with append-only constraint)
- **Potential Harm/Impact:** Any caller (orchestrator, Skills API) can write personal data into the `payload` column of the append-only table; once written it cannot be deleted without custom redaction infrastructure; GDPR erasure requests cannot be fulfilled for data embedded in append-only audit events
- **Ability to Implement Control:** Medium
- **Recommended controls:**
  - Add a payload schema validator or allow-list of permitted payload keys per `event_type`; reject or strip keys not on the allow-list before insert.
  - Implement a selective in-place redaction mechanism (`UPDATE audit_events SET payload = redacted_payload WHERE event_id = ?`) accessible only to a privileged GDPR admin role — does not delete the row, replaces personal content with a redaction marker and records redaction timestamp and operator.
  - Document all permitted payload structures per event type in the data dictionary and enforce via code review gate.

### Risk 3: Frontend CSV export creates uncontrolled copies of telemetry containing personal data

- **Priority:** High
- **Risk Category:** Data exfiltration and GDPR Art. 32 security of processing
- **GDPR Reference:** Art. 32 — security of processing; Art. 5(1)(f) — integrity and confidentiality
- **Potential Harm/Impact:** Authenticated users can download prompt/output pair data to their local device; the file is outside all system access controls, audit logs, and retention enforcement from that point forward; personal data in prompts/outputs can persist on unmanaged endpoints after the system record is deleted or redacted
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Apply the same pre-export redaction as recommended in Risk 1: serve redacted prompt/output pairs to the export endpoint.
  - Add an audit event (`observability.csv_export`) recording who exported, what time window, and how many records — this creates an accountability trail even if the file cannot be tracked.
  - Consider gating CSV export to a specific high-privilege role (e.g., `qm_lead` only, not `auditor` or service clients).

### Risk 4: Prometheus `/metrics` endpoint exposed without authentication

- **Priority:** Medium
- **Risk Category:** Operational data exposure and access control gap
- **GDPR Reference:** Art. 25 — data protection by design; Art. 32 — appropriate technical measures
- **Potential Harm/Impact:** The `/metrics` endpoint exposes request counts by path and status code (`dq_http_requests_total` labels: `method`, `path`, `status`). Path-level cardinality in Prometheus is normalised by UUID and integer scrubbing in `observability.py`, but metric label values could still expose usage patterns (which document routes are active, error rates) to any unauthenticated network caller
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Gate `/metrics` behind network-level access control (restrict to monitoring subnet / Kubernetes namespace only).
  - Alternatively, add a bearer-token check (`METRICS_BEARER_TOKEN` env var) before returning the response body.
  - Verify that path normalisation in `_UUID_RE` and `_INT_RE` fully removes document or session IDs from metric label values before they are stored in the Prometheus time series.

### Risk 5: OTEL trace data sent to external OTLP endpoint without documented Data Processor Agreement

- **Priority:** Medium
- **Risk Category:** Cross-boundary data transfer and third-party processing
- **GDPR Reference:** Art. 28 — data processor; Art. 46 — transfers to third countries (if OTLP endpoint is SaaS)
- **Potential Harm/Impact:** When `TRACING_EXPORTER=otlp` is configured, OTEL spans (including HTTP path, method, status, user-agent, trace context) are sent to an external endpoint; this constitutes a GDPR data transfer to a processor/sub-processor without a documented agreement; sampling ratio defaults to `1.0` (full capture of every request)
- **Ability to Implement Control:** Medium
- **Recommended controls:**
  - Default `TRACING_EXPORTER` to `none` (or `console`) in production until a Data Processor Agreement with the OTLP backend is in place and reviewed.
  - Reduce default `TRACING_SAMPLING_RATIO` to `0.1` or lower for production to limit volume of exported span data.
  - Strip or hash path-embedded identifiers (document IDs, session IDs) from span attributes before export using an OTEL `SpanProcessor` attribute scrubber.
  - Document the OTLP endpoint vendor in the GDPR Record of Processing Activities (Art. 30).

### Risk 6: `agent_telemetry` retention policy is defined conceptually but not enforced in schema or application layer

- **Priority:** Medium
- **Risk Category:** Data retention and governance completeness
- **GDPR Reference:** Art. 5(1)(e) — storage limitation; Art. 25 — data protection by design
- **Potential Harm/Impact:** The persistence Definition of Done distinguishes `agent_telemetry` (short retention, non-compliance-critical) from `audit_events` (long retention); however, no `agent_telemetry` table definition, TTL enforcement, or purge job was found in the observed schema — meaning operational telemetry may be silently accumulating in `quality_observations` with no enforced deletion
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Implement the `agent_telemetry` table and purge job as defined in the persistence DoD (Phase 0).
  - Alternatively, add a `retention_class` column to `quality_observations` (`operational` vs `audit_evidence`) and run a scheduled job to delete rows where `retention_class = 'operational' AND event_time < NOW() - INTERVAL '90 days'`.
  - Define explicit retention periods in the GDPR Record of Processing Activities and document the enforcement mechanism.

## 5. Cross-Sheet Consistency

| Control Area | Related Risk Sheet | Alignment Required |
| --- | --- | --- |
| `quality_observations.payload` redaction | Risk Sheet 1 (Model Providers, Risk 2) | Same redaction policy must apply whether traces are triggered by external or on-prem model calls |
| Observability API service-client access | Risk Sheet 2 (RBAC, Risk 5) | Service-client access to `llm-traces` must serve only redacted payload; must be access-logged |
| `audit_events` append-only GDPR erasure conflict | Risk Sheet 2 (RBAC, Risk 3) | Access-decision audit log and the compliance audit trail face the same erasure-vs-retention tension — same selective redaction solution applies |
| OTEL exporter data processor agreement | Risk Sheet 1 (Model Providers) | OTLP backend must appear in the same sub-processor register as external model API providers |
| CSV export — uncontrolled copy | Risk Sheet 4 (Secrets/Tokens — pending) | Exported files containing model outputs should be treated under the same data handling classification as API responses containing model output |
