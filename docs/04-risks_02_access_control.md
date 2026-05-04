# AI System Privacy Audit: Role-Based Access Controls and GDPR Compliance

System in scope: `doc_quality_compliance_check` (backend API, session auth, RBAC layer, PostgreSQL user/session storage).

## 1. System Diagram

![](images/raw/compliance-checker-2026-05-02.png)

RBAC-relevant architecture facts used in this risk sheet:

- The backend exposes a FastAPI application with two authentication modes: browser users via HTTP-only session cookies (email/password login) and service clients via `X-API-Key` / `Authorization: Bearer` header.
- Role enforcement uses a `require_roles(...)` dependency on every protected route. Roles defined: `qm_lead`, `architect`, `riskmanager`, `auditor`, `service`.
- The `service` role is restricted to explicitly machine-to-machine routes (`/api/v1/skills/*`, `/api/v1/observability/*`); it is no longer a blanket bypass.
- Two routes are **currently unauthenticated**: `/api/v1/dashboard/*` and `/api/v1/templates/*`.
- Session data (email, roles, org, `last_seen_at`) is persisted to PostgreSQL. Bootstrap/MVP credentials are configured via environment variables.

## 2. Data Flow Analysis

| Data Flow | Source | Destination | Encrypted? | Logged? | Priority |
| --- | --- | --- | --- | --- | --- |
| Login request (email + password) | Browser / API client | `POST /api/v1/auth/login` → session store (PostgreSQL) | In-transit (HTTPS/TLS) | Auth event (success/failure, timestamp, email); login throttle state | High |
| Session cookie issued to browser | FastAPI auth route | Browser (HTTP-only `Set-Cookie`) | In-transit (TLS); cookie is HTTP-only, `secure` flag enforced outside dev | Session record in PostgreSQL (token hash, email, roles, org, expiry) | High |
| Authenticated request (cookie or API key) | Browser / service client | FastAPI route → `require_roles` dependency | In-transit (HTTPS/TLS) | `last_seen_at` updated in session row; access control outcome not separately audit-logged | High |
| Service-client auth (API key / Bearer) | Orchestrator or automation tool | FastAPI `/skills/*` or `/observability/*` endpoints | In-transit (HTTPS/TLS) | API key check in `require_api_auth`; no per-call access log entry | High |
| Unauthenticated dashboard access | Any caller (no credentials required) | `GET /api/v1/dashboard/*` | In-transit (HTTPS/TLS) | No auth check; no identity logged | Medium |
| Unauthenticated template access | Any caller (no credentials required) | `GET /api/v1/templates/*` | In-transit (HTTPS/TLS) | No auth check; no identity logged | Medium |
| Bootstrap user provisioning | Environment config (`AUTH_MVP_*` vars) | FastAPI startup → `app_users` table | Config-level (env vars / secrets manager) | Startup event; credential in env var outside runtime log | High |
| Role and org resolution (session row) | PostgreSQL session table | `resolve_user_from_cookie` / `require_roles` | At-rest DB controls | Implicit (session row lookup); no dedicated access-decision log | Medium |
| Session revocation on logout | `DELETE /api/v1/auth/logout` | PostgreSQL (`is_revoked = True`) | In-transit (HTTPS/TLS) | Logout event in session row state; no separate audit-trail entry | Medium |

### Corrected interpretation for RBAC and GDPR

- The primary GDPR boundary in the auth flow is the persistence of identity data (`user_email`, `user_org`, `user_roles`, `last_seen_at`) in the session table — minimisation and retention obligations apply.
- The `viewer` role appears in tests (`test_auth_authorization_api.py`) but is **not** in the documented implemented role set. A `viewer` caller receives `403` on all tested routes, meaning this role effectively has zero access — a potential misconfiguration gap.
- The `service` role hardening (no blanket bypass) is a positive control. Residual risk: the `observability/*` endpoint remains service-accessible and can return rich trace payloads containing personal data (see also Risk Sheet 1 — model trace over-collection).
- The two unauthenticated routes expose application data without identity context, which is inconsistent with GDPR accountability and access-control principle (GDPR Art. 5(1)(f), Art. 25).

## 3. Sensitive Data

### Sensitive Data: User Identity in Session Store

- **Category:** Personal data (identified natural persons) — GDPR Art. 4(1)
- **Examples:** `user_email` (primary identifier), `user_org`, session `session_id`, `expires_at`, `last_seen_at`
- **Why Sensitive:** Directly identifies users and links their organisational role to access patterns and audit trails; retained in PostgreSQL with no visible TTL-based purge policy
- **Current Protection:** Server-side session with hashed token; DB access controls; session expiry and revocation support
- **Risk (or Harm) if Exposed:** Unauthorised disclosure of user identity and role assignments; GDPR breach; profiling of users from access patterns

### Sensitive Data: Role Assignments and Permission Scope

- **Category:** Attributes linked to natural persons — implicit GDPR personal data when combined with identity
- **Examples:** `user_roles` array per session row (e.g., `["qm_lead"]`, `["auditor"]`), org isolation field `user_org`
- **Why Sensitive:** Reveals organisational responsibilities and access privileges; can be used for social engineering or targeted attacks; GDPR data minimisation applies
- **Current Protection:** Stored in session row; resolved per request by `require_roles`; role set validated at route boundary
- **Risk (or Harm) if Exposed:** Privilege mapping; lateral movement by attacker with partial DB access; GDPR accountability gap if role-to-action mapping is not audited

### Sensitive Data: Bootstrap/MVP Credentials in Environment Configuration

- **Category:** Authentication secrets and provisioning data
- **Examples:** `AUTH_MVP_EMAIL`, `AUTH_MVP_PASSWORD`, `AUTH_MVP_ROLES`, `AUTH_MVP_ORG` (env vars); `SECRET_KEY` (API key secret)
- **Why Sensitive:** Compromise of bootstrap credentials gives attacker a fully provisioned account with configurable roles; `SECRET_KEY` grants service-client access to `skills/*` and `observability/*`
- **Current Protection:** Environment-variable configuration (not in code); excluded from source code
- **Risk (or Harm) if Exposed:** Full account takeover; unrestricted access to trace data; GDPR breach; credential reuse risk if same password is used across environments

### Sensitive Data: Access Decision and Audit Context Not Separately Persisted

- **Category:** Audit/traceability gap — GDPR Art. 5(2) accountability
- **Examples:** Which role accessed which route at what time; 403 access denials; service-client route usage with payload summary
- **Why Sensitive:** Absence of access-decision audit log prevents retrospective investigation of data-access incidents; required for GDPR Art. 30 Record of Processing Activities and breach response
- **Current Protection:** `last_seen_at` updated on session lookup (coarse-grained); login throttle state tracked per email/IP
- **Risk (or Harm) if Exposed:** Inability to detect or evidence unauthorised access; weakened GDPR breach-response capability

## 4. Privacy Risks

### Risk 1: Unauthenticated routes expose application data without identity context

- **Priority:** High
- **Risk Category:** Access control — missing authentication
- **GDPR Reference:** Art. 5(1)(f) — integrity and confidentiality; Art. 25 — data protection by design
- **Potential Harm/Impact:** Any network-accessible caller can read dashboard data and templates without establishing identity; GDPR accountability principle violated (no record of who accessed what); potential leak of structural compliance artefacts or meta-information
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Add `require_authenticated_user` dependency to all `/api/v1/dashboard/*` and `/api/v1/templates/*` routes.
  - If anonymous read-only access is intentional for templates, scope it to non-personal, non-compliance-sensitive data only and document the deliberate design choice in the privacy notice.
  - Add access logging (caller IP or session ID + timestamp) for these routes until auth is enforced.

### Risk 2: Session table retains personal data (email, org, last_seen_at) without visible purge policy

- **Priority:** High
- **Risk Category:** Data retention and minimisation — GDPR Art. 5(1)(e) and Art. 25
- **GDPR Reference:** Art. 5(1)(e) — storage limitation; Art. 13/14 — data subject transparency on retention
- **Potential Harm/Impact:** Expired or revoked sessions (`is_revoked = True`) remain in `app_user_sessions` with full personal data indefinitely; contradicts GDPR storage limitation principle; complicates data-subject erasure requests (Art. 17)
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Implement a scheduled job (e.g., nightly) to hard-delete session rows where `is_revoked = True OR expires_at < NOW() - <grace_period>`.
  - Define and document maximum retention period for session records (e.g., 30 days post-expiry).
  - On GDPR erasure request, immediately revoke and delete all session rows for the subject's email.

### Risk 3: No dedicated access-decision audit log (GDPR accountability gap)

- **Priority:** High
- **Risk Category:** Audit trail and accountability — GDPR Art. 5(2), Art. 30
- **GDPR Reference:** Art. 5(2) — accountability; Art. 32 — security of processing; Art. 33 — breach notification readiness
- **Potential Harm/Impact:** `last_seen_at` on session lookup provides coarse activity signal but does not record which route was accessed, what role was used, or whether a 403 was returned; in a breach scenario, no evidence trail exists for forensic investigation or Data Protection Authority reporting
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Add a FastAPI middleware or route-level dependency that writes structured access-decision log entries: `session_id`, `email`, `roles`, `method`, `path`, `status_code`, `timestamp`.
  - Include 403 denial events in the log (route attempted, role required vs role held).
  - Protect this log with the same retention and access controls as the main audit-trail table.
  - Cross-reference: access-decision log entries should link to `audit_events` `correlation_id` where available.

### Risk 4: `viewer` role defined in test code but absent from authorised role set creates misconfiguration risk

- **Priority:** Medium
- **Risk Category:** RBAC gap and role lifecycle governance
- **GDPR Reference:** Art. 25 — data protection by design; Art. 32 — appropriate technical measures
- **Potential Harm/Impact:** The `viewer` role returns `403` on all tested routes but is not defined in the canonical role registry. If a user is assigned `viewer` (e.g., through bootstrap config or future user management), they receive zero access — which may be the intended behaviour but is undocumented. Alternatively, if a future route is added and `viewer` is not excluded, it could gain unintended access
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Define an explicit canonical role registry (enum or constants file) listing all valid roles with their intended permission scope and whether they are human or machine roles.
  - Add a startup assertion or test that no undefined role names can be provisioned via bootstrap config.
  - Document whether `viewer` is a planned future role (read-only access) or should be removed entirely.

### Risk 5: Service-client (`X-API-Key`) can access observability endpoints containing personal data in traces

- **Priority:** Medium
- **Risk Category:** Least-privilege and machine-identity access to personal data
- **GDPR Reference:** Art. 25 — data protection by design (least privilege); Art. 32 — appropriate technical measures
- **Potential Harm/Impact:** The `observability/*` endpoint is explicitly marked `allow_service=True`, meaning automated orchestrator or any bearer-token holder can retrieve rich trace payloads that may contain personal data from documents and prompts (see Risk Sheet 1, Risk 2). Service clients have no session identity or org context, making attribution difficult
- **Ability to Implement Control:** Medium
- **Recommended controls:**
  - Apply pre-response redaction to observability payloads served to service clients: strip `prompt_text`, `output_text`, and any fields flagged as personal-data-bearing before returning.
  - Log service-client access to observability endpoints with `X-API-Key` identifier hash and query parameters in the access-decision log.
  - Consider restricting service-client access to operational metadata only (latency, status, counts), not the rich trace payload.

### Risk 6: Bootstrap/MVP credential pattern must not persist to production

- **Priority:** Medium
- **Risk Category:** Credential management and production hardening
- **GDPR Reference:** Art. 32 — appropriate technical measures for security of processing
- **Potential Harm/Impact:** If `AUTH_MVP_EMAIL` / `AUTH_MVP_PASSWORD` env vars are reused or not rotated between environments, a staging credential compromise grants production access; bootstrap accounts may accumulate active sessions that are never revoked; default password (`CHANGE_ME_BEFORE_USE` visible in test code) presents brute-force risk if left in place
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Enforce a startup check: if `AUTH_MVP_ENABLED=true` and environment is `production`, reject startup unless password has been explicitly changed from default.
  - Document the bootstrap account as a break-glass account: rotate immediately after initial provisioning, restrict to a dedicated `bootstrap` role with minimal scope, and disable after handover.
  - Use a secrets manager (e.g., Vault, AWS Secrets Manager) for all production credential injection rather than plain environment variables.

## 5. RBAC Consistency with Risk Sheet 1 (Model Providers)

The following cross-cutting controls are shared with the model-provider risk set and must remain consistent:

| Control Area | Risk Sheet 1 Finding | Risk Sheet 2 Alignment Required |
| --- | --- | --- |
| Observability endpoint access | Over-retained trace data with personal data | Service-client access to observability must apply same redaction policy (sheet 1, Risk 2) |
| Audit trail completeness | Model/version metadata per run | Access-decision log must link to same `correlation_id` as audit events |
| Least-privilege principle | Prompt/output access restricted by role | Role enforcement on observability and skills endpoints must not be wider than role enforcement on document/compliance routes |
| HITL approval for high-impact outputs | HITL role required for approval actions | HITL-approver role must map to a defined RBAC role (`qm_lead` or `auditor`) with explicit route permission — no anonymous or service-client approval permitted |

---

## Additional information from the repo

### What exists

- **RBAC:** `require_roles(...)` gates API routes (e.g. documents, skills, research).
- **Cookies:** Session cookie is `HttpOnly`, `SameSite=lax`, `Secure` enforced outside development (`session_auth.py`, `config.py`).
- **`AuthenticatedUser`** includes `org` from session (`user_sessions.user_org`).
- **`audit_events`**, **`audit_schedules`**, **`LogEventRequest`**: optional `tenant_id` / `org_id` fields for labeling.

### document and HITL scope

- **`SkillDocumentORM`** has **no** `org_id` or `tenant_id` column.
- **`search_documents`** (`src/doc_quality/services/skills_service.py`) queries **all** `skill_documents` rows (filters: type, optional text search against `extracted_text` / filename), capped by `limit`.
- **`GET /api/v1/documents`** uses `search_documents` with empty query → returns up to 100 documents for **any** authenticated user with an allowed role — **no filter by `user.org`**.
- **HITL** (`hitl_workflow.py`): no org/tenant parameters in queries.

**Conclusion:** The codebase matches a **single-tenant / shared-database** MVP: isolation is **role-based**, not organization-row-level. Multi-customer SaaS would need schema + query changes and consistent propagation of org from JWT/session into every read/write path.
