# Use cases and risk areas to investigate

## 1. Areas to explore for privacy risks

- Personal/sensitive data storage 
- Egress to openclaw orchestrator and models (perplexity, anthropic, others) (Prio 1)
- Export of reports to storage
- Role base access controls (Prio 2)
- Telemetry: evaluation user workflow tracking (Prio 3)
- Structured logging: coding workflow tracking (Prio 3)
- Secrets, tokens, and keys (Prio 4)
- CI

## 2. General application workflow

- User interacts with application
- User logs in
- User uploads document
- Document is persisted in DB
- Document is analyzed by AI
   - AI suggests risk templates
   - AI suggests document analysis
   - AI suggests human review
- Human review process HITL (Right now always the same user, because it's connected to the login. May change in the future.)
- Store reviewed document
- Store reports
- Export reports to recipients

## 3. General software processes and services

- Access controls (RBAC)
- Authentication
- Authorization
- Logging
- Telemetry
- Secrets, keys, and tokens
- CI/CD
- Docker containerization

## 4. Data storage

### Database tables for application (ORM: `src/doc_quality/models/orm.py`)

- `app_users`
- `user_sessions`
- `password_recovery_tokens`
- `skill_documents` 
- `hitl_reviews`
- `bridge_human_reviews`
- `audit_events`
- `audit_schedules`
- `skill_findings`
- `quality_observations`
- `stakeholder_employee_assignments`
- `risk_templates`
- `document_locks`

### Auth-related audit payloads (`src/doc_quality/api/routes/auth.py`)

- Logins
- password recovery
- cookies
- session tokens

### Structured logging

- Application logs
    - errors
    - requests
    - status events
- Spans for tracing
- Document skills output persisted
- Access logs
- AI services: document excerpts, errors, events, and logs

## 5. AI orchestrator and models — Anthropic and Perplexity

### OpenClaw 

- Orchestrate messages to AI agents

### Perplexity

- Requests
- Prompts
- Responses
- Metadata
- Payloads

### Anthropic

- Requests
- Prompts
- Responses
- Metadata
- Payloads

Agents:
- Document analysis agent (doc_check_agent)
- Compliance agent (compliance_agent)

Risk mitigation: 
- in example template ->>> to be removed

## 6. Exports, remote uploads, records retention

### Export registry

- local downloads
- remote exports

### Records retention

Who has access to what?
How long?

- local
- offsite

Risk mitigation:
- RBAC 
- admin account can define roles and access
- storage is forever right now
- admin can delete users and records

### IDE / MCP / Copilot

What is used

- VS Code & GitHub Copilot

## 7. Access controls (individuals / org)

- RBAC
- Authenticated users and HITL reviewers
- Audit logs

### Document and Human in the Loop (HITL)

- Document access
- HITL review access and actions

### Secrets, tokens, and keys (`src/doc_quality/core/config.py`)

- Secret key for application
- DB access
- Anthropic and Perplexity API keys
- Auth logins
- Tokens
- Cookies

## 8. Logging, configuration, telemetry, CI, Docker

### Logging

Structured logging and information collected.

### Telemetry and metrics

- Traces and spans stored
- Data collection by Prometheus
- Metrics exposed to endpoints

### Docker

Container logs and info to stdoout or stderr

### CI

- action artifacts
- testing artifacts, like Playwright screenshots

## 9. Documentation and dev data

- Snippets of real data
- Mocked data for testing
