# AI System Privacy Audit: Structured Logging

System in scope: `doc_quality_compliance_check` — structlog-based application logging layer, HTTP request middleware log output, service-level log events, and the relationship between application logs and the persisted `audit_events` compliance trail.

## 1. System Diagram

![](images/raw/compliance-checker-2026-05-02.png)

Structured-logging-relevant architecture facts used in this risk sheet:

- The application uses **structlog ≥ 24.1.0** configured via `core/logging_config.py`, with `PrintLoggerFactory` writing to stdout in JSON (production) or console (development) format.
- Log format is controlled by `LOG_FORMAT` env var (`json` for production); log level by `LOG_LEVEL` (default `INFO`).
- The **HTTP request middleware** in `api/main.py` binds `request_id`, `correlation_id`, and `trace_id` as context variables and logs every request with `method`, `path`, `status`, `duration_ms`.
- Service-level loggers (`get_logger(__name__)`) emit structured events at key decision points: HITL review lifecycle, report generation, research requests, template loading.
- The **auth route** writes `auth.login_success` / `auth.login_failure` events to `audit_events` (PostgreSQL) via the `log_event` service — not to the structlog stream — but login-related events appear in both surfaces.
- Rate-limiting signals (login throttle, recovery throttle) are tracked in-memory per email/IP; abuse events are logged.
- **No PII redaction processor** is present in the structlog processor chain (`shared_processors` contains only `merge_contextvars`, `add_log_level`, `TimeStamper`, and a renderer).
- Structlog output goes to stdout; downstream retention, rotation, and access control depend entirely on infrastructure (e.g., Docker log driver, Kubernetes log aggregator, cloud logging service).

## 2. Data Flow Analysis

| Data Flow | Source | Destination | Encrypted? | Logged? | Priority |
| --- | --- | --- | --- | --- | --- |
| HTTP request log entry emitted per request | FastAPI middleware | stdout (structlog) → log aggregator | Depends on infrastructure transport | Every request: `method`, `path`, `status`, `duration_ms`, `request_id`, `correlation_id`, `trace_id` | High |
| Auth login event (success / failure) | `POST /api/v1/auth/login` | `audit_events` PostgreSQL table **and** structlog stream | In-transit + at-rest DB controls | Email visible as `actor_id` in `audit_events`; login outcome logged in structlog | High |
| Rate-limit / lockout event | `core/rate_limit.py`, auth route | structlog stream | Depends on infrastructure | IP address and email appear as identifiers in throttle/lockout events | High |
| Service lifecycle events (HITL, reports, research) | Service layer (`hitl_workflow.py`, `report_generator.py`, `research_service.py`) | structlog stream | Depends on infrastructure | `review_id`, `document_id`, `report_id`, `domain`, `model`, `error` fields in log lines | Medium |
| Research API error with domain and model | `research_service.py` | structlog stream | Depends on infrastructure | `domain`, `model`, `error`, `status_code` logged on Perplexity API failure — domain may be business-sensitive | Medium |
| Application startup / shutdown | `api/main.py`, orchestrator `main.py` | structlog stream | Depends on infrastructure | `app_version`, `environment` logged; no credential values in default config | Low |
| DEBUG-level log output (if `LOG_LEVEL=DEBUG`) | Any service logger | structlog stream | Depends on infrastructure | May include full request context, raw Pydantic model dumps, provider SDK debug output containing prompts/responses | High |
| structlog output to log aggregator / SIEM | stdout (container runtime) | Log aggregator (e.g., Loki, CloudWatch, Datadog) | Depends on infrastructure configuration | Entire log stream; third-party processor if SaaS aggregator — GDPR Art. 28 applies | High |

### Corrected interpretation for GDPR

- The HTTP middleware logs `path` for every request — document paths like `/api/v1/documents/doc-abc` embed document identifiers; stakeholder paths embed profile IDs. These are personal-data-adjacent fields under GDPR if combined with a timestamp and user identity.
- The structlog stream does **not** log user identity on regular requests (no `user_email` field in the HTTP middleware), which is a positive privacy-by-design property. However, service-level loggers emit `reviewer_email` or `actor_id` in specific events.
- IP address appears implicitly in rate-limit and lockout enforcement — whether it is emitted to the log stream needs verification; IP is personal data under GDPR (CJEU *Breyer* ruling).
- If a SaaS log aggregator is used (Datadog, Splunk Cloud, CloudWatch), the entire log stream — including any personal data fields — constitutes a transfer to a data processor, requiring a GDPR Art. 28 Data Processing Agreement.

## 3. Sensitive Data

### Sensitive Data: User Email in `actor_id` of Auth Audit Events

- **Category:** Personal data (identified natural person) — GDPR Art. 4(1)
- **Examples:** `actor_id = "user@example.com"` in `auth.login_success`, `auth.login_failure`, `auth.recovery.requested` events written to `audit_events` and referenced in structlog entries
- **Why Sensitive:** Directly identifies users; combined with `event_type`, `event_time`, and `payload.roles` creates a profile of user authentication behaviour; stored in the long-retention audit trail
- **Current Protection:** `audit_events` table is role-gated; structlog output to stdout (downstream controls depend on infrastructure)
- **Risk (or Harm) if Exposed:** Profiling of user login frequency and role assignments; breach of GDPR Art. 5(1)(f) confidentiality if log stream is accessible beyond authorised operators

### Sensitive Data: IP Address in Rate-Limit and Lockout Events

- **Category:** Online identifier — GDPR Art. 4(1); personal data per CJEU *Breyer* ruling
- **Examples:** IP address used as key in per-IP login throttle (`auth_login_rate_limit`) and recovery throttle; emitted as identifier in abuse-detection log entries
- **Why Sensitive:** IP addresses are personal data under GDPR; stored in-memory rate-limit state and potentially emitted to log stream; if logged to a persistent aggregator, retention must comply with data minimisation obligations
- **Current Protection:** In-memory rate-limit state (no DB persistence observed); log output depends on infrastructure
- **Risk (or Harm) if Exposed:** GDPR breach if IP addresses are logged to long-retention aggregators without a legal basis and retention policy; enables correlation of individual users across sessions via IP linkage

### Sensitive Data: Document and Entity Identifiers in HTTP Path Log Fields

- **Category:** Indirect personal data (pseudonymous identifiers linkable to persons)
- **Examples:** `/api/v1/documents/{document_id}`, `/api/v1/stakeholders/{profile_id}`, `/api/v1/bridge/{run_id}` — all logged as `path` in every HTTP request entry
- **Why Sensitive:** Document IDs and stakeholder profile IDs are pseudonymous references to personal data records; combined with timestamp and source IP they can reconstruct a user's activity trail
- **Current Protection:** HTTP log entry does not include user email or session ID (positive control); HTTPS in transit
- **Risk (or Harm) if Exposed:** Re-identification of user activity from log records; access pattern reconstruction; GDPR Art. 5(2) accountability gap if logs are not access-controlled

### Sensitive Data: Service-Level Log Fields with Reviewer Identity

- **Category:** Personal data embedded in operational log events
- **Examples:** `reviewer_email` in HITL workflow log entries (`review_created`, `review_status_updated`); `actor_id` (email) in Skills API skill event log; `domain` in research service error logs
- **Why Sensitive:** Reviewer email directly identifies a natural person and is logged alongside document IDs and review decisions; persists in log stream for as long as logs are retained
- **Current Protection:** Structlog output to stdout; no PII redaction processor in chain; retention and access depend on infrastructure
- **Risk (or Harm) if Exposed:** Reviewer identity linked to specific document review decisions in log records; GDPR violation if logs are retained beyond operational need or shared with third-party aggregators without a DPA

## 4. Privacy Risks

### Risk 1: No PII redaction processor in structlog chain — personal data emitted in plaintext

- **Priority:** High
- **Risk Category:** Logging minimisation and PII scrubbing
- **GDPR Reference:** Art. 5(1)(c) — data minimisation; Art. 25 — privacy by design; Art. 32 — security of processing
- **Potential Harm/Impact:** Emails, document IDs, and reviewer identifiers flow through the structlog processor chain without any scrubbing or pseudonymisation; any consumer of the log stream (log aggregator, on-call engineer, third-party SIEM) receives plaintext personal data; no technical mechanism prevents accidental inclusion of additional sensitive fields in future log entries
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Add a custom structlog processor (between `TimeStamper` and renderer) that redacts or pseudonymises known sensitive field names: `reviewer_email`, `actor_id` (if email), `user_email`, `email`, and optionally `actor_id` values matching an email pattern.
  - Publish a list of permitted log field names in the project coding standards; add a linting rule or test that checks log call sites do not introduce new unlisted PII fields.
  - For audit-grade events that require the email (e.g., `auth.login_success`), route them exclusively to `audit_events` (DB, role-gated) rather than the structlog stream.

### Risk 2: DEBUG log level may expose full request context, model outputs, and provider SDK traces

- **Priority:** High
- **Risk Category:** Log-level governance and environment hardening
- **GDPR Reference:** Art. 5(1)(c) — data minimisation; Art. 32 — security of processing
- **Potential Harm/Impact:** `LOG_LEVEL=DEBUG` is the natural development setting; if accidentally applied to staging or production it exposes full Pydantic model dumps, raw provider API responses (which may contain prompt/output content), and internal service state including personal data; this has occurred in real incidents at comparable systems
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Add a startup assertion: if `environment == "production"` and `log_level == "DEBUG"`, fail application startup with a clear error message.
  - Default `LOG_LEVEL` to `WARNING` or `INFO` in production environment configuration; require an explicit override with a documented justification for any temporary `DEBUG` activation in production.
  - Ensure provider SDK (Anthropic, Perplexity) debug output is suppressed independently of the application's log level by setting their respective logger levels explicitly to `WARNING` in `configure_logging()`.

### Risk 3: Log output destination and retention are fully infrastructure-delegated with no application-level policy

- **Priority:** High
- **Risk Category:** Log retention governance and GDPR storage limitation
- **GDPR Reference:** Art. 5(1)(e) — storage limitation; Art. 30 — record of processing; Art. 28 — data processor (if SaaS aggregator)
- **Potential Harm/Impact:** `PrintLoggerFactory` writes to stdout; all retention, rotation, access control, and deletion are delegated to whatever log aggregator is configured at deployment time; there is no application-level policy that says "structured logs must not be retained for more than N days" or "access to log streams is restricted to operator role Y"; if a SaaS aggregator is used (Datadog, Splunk Cloud, CloudWatch), the entire log stream is processed by a third party without a mandatory GDPR Art. 28 agreement in the current implementation
- **Ability to Implement Control:** Medium
- **Recommended controls:**
  - Define a maximum log retention period (e.g., 30 days for operational logs; `audit_events` DB trail serves long-term compliance needs) in the deployment runbook and enforce it in the log aggregator configuration.
  - Document the log aggregator vendor(s) in the GDPR Record of Processing Activities as data processors; ensure a signed DPA is in place before sending logs to any SaaS service.
  - Add log stream access control to the infrastructure design: restrict log query access to the same roles that can access the `audit_events` API (`qm_lead`, `auditor`, `riskmanager`).

### Risk 4: IP address handling in rate-limit events is undocumented for GDPR purposes

- **Priority:** Medium
- **Risk Category:** Online identifier data governance
- **GDPR Reference:** Art. 4(1) — definition of personal data (IP as online identifier); Art. 5(1)(e) — storage limitation
- **Potential Harm/Impact:** Login and recovery rate-limit logic uses IP address as a throttle key; if the IP is included in structlog event fields (e.g., in a `client_ip` or `remote_addr` field on abuse events), it becomes part of the log stream and is subject to GDPR; in-memory storage means no persistence but also no audit of abuse patterns; unclear if IP is emitted to persistent log aggregator
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Review all log call sites in `auth.py` and `rate_limit.py` to confirm whether `client_ip` or equivalent is emitted; if so, hash it (SHA-256 with a service-specific salt) before logging.
  - Document the IP address processing in the GDPR Record of Processing Activities: purpose (abuse prevention), legal basis (legitimate interest), retention (in-memory only — no persistence), and minimisation measure (hashed if logged).

### Risk 5: Structlog and `audit_events` serve overlapping but inconsistent purposes — risk of duplication and compliance confusion

- **Priority:** Medium
- **Risk Category:** Audit trail governance and data duplication
- **GDPR Reference:** Art. 5(2) — accountability; Art. 30 — record of processing
- **Potential Harm/Impact:** Some events (e.g., login success, review lifecycle) appear in both the structlog stream and `audit_events` PostgreSQL table. This creates two sources of truth with different schemas, retention periods, and access controls; inconsistency between them complicates GDPR breach investigation, subject access requests, and external audit responses; structlog entries may contain more detail than `audit_events` or vice versa
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Define a clear event routing rule: compliance-critical events (auth, document actions, review decisions, workflow completions) go exclusively to `audit_events` (DB, structured, role-gated); operational events (latency, retry, routing mode, startup) go to structlog stream only.
  - Remove or suppress the structlog emission of events that are already persisted to `audit_events` to eliminate dual-stream confusion.
  - Document the routing rule in the OBSERVABILITY_LOGGING_README and enforce via a code review checklist item.

## 5. Cross-Sheet Consistency

| Control Area | Related Risk Sheet | Alignment Required |
| --- | --- | --- |
| PII in log fields | Risk Sheet 3 (Telemetry) | Same redaction-processor approach applies to both structlog stream and pre-write quality observation service |
| Log aggregator SaaS DPA | Risk Sheet 3 (Telemetry) | OTLP backend and log aggregator must both appear in the GDPR Art. 28 processor register |
| `audit_events` dual-write from structlog | Risk Sheet 2 (RBAC, Risk 3) | Access-decision log (recommended in sheet 2) should route to `audit_events`, not to the structlog stream, to benefit from role-gated access |
| DEBUG mode in production | Risk Sheet 1 (Model Providers) | DEBUG output from provider SDKs could expose full prompt/output content — startup guard must cover SDK-level loggers as well |
| IP address logging | Risk Sheet 2 (RBAC, Risk 4) | Bootstrap credential brute-force protection relies on IP throttle; IP handling must be consistent across auth and rate-limit code paths |
