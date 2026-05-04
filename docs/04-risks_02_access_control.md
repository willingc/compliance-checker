# Risk level 2: Role based access controls (RBAC)

**AI System Privacy Audit: Documentation Quality Compliance Checker**

!!! note Priority 3

    Role based access controls (RBAC)

## 1. System Diagram

![](images/raw/compliance-checker-2026-05-02.png)

## 2. Data Flow Analysis

| Data Flow | Source | Destination | Encrypted? | Logged? | Priority |
|-----------|--------|-------------|------------|---------|---------|
| User input | Web app | LLM API | In-transit | Yes | High |
| Model output | LLM API | Database | No | 50% | Medium |

## 3. Sensitive Data

### Sensitive Data Name: Access to data based on a user's role

- **Category:** Document contents, database records
- **Examples:** informmation about people contained in documents
- **Why Sensitive:** May contain data about individuals, document processes, quality information, potential reputation risk of company or individuals
- **Current Protection:** Role based on the user's need to access functions and tasks
- **Risk (or Harm) if Exposed:** Data breach; personal data leaks

## 4. Privacy Risks

### Risk 2: Role based access controls RBAC

- **Priority:** [High]
- **Risk Category:** [other - permissions and access to data)]
- **Potential Harm/Impact:** A malicious actor could access any information stored in database and documents
- **Ability to Implement Control:** [Medium]
- **Recommended controls:**
  - Use a well-vetted library for role based access controls
  - Implement only enough privilege to do the needed work
  - Time bound any privilege that is only needed for a single task
  - Rigorously remove any individuals who no longer require access

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
