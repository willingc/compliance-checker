---
marp: true
theme: rose-pine
footer: "AI Privacy Capstone - May 2026"
paginate: true
---

<!-- _class: title -->

![bg right:48% contain](images/raw/Docquality_LoginPage.png)

# Documentation Quality Compliance Checker

<p class="title-authors">
  <span>Ilona Brinkmeier</span>
  <span>Wiebke Meyer</span>
  <span>Carol Willing</span>
</p>

---

<!-- class: sec1 -->
<!-- _class: sec1 divider -->

# Project & use case

- What we audited
- Problem solved
- Architecture at a glance
- Why privacy matters

---

![bg left:50% contain](images/PII-CollectionAndPrivacy-2026-04-26.svg)

### What we audited

- **Scope:** Documentation quality compliance checker performs AI-assisted QM audits of technical docs and governance docs (SOPs)
- **Repo:** https://github.com/IloBe/doc_quality_compliance_check
- **Audit:** https://willingc.github.io/compliance-checker/

<!--
## 1. Introduce your Use Case (~2 min)

Goal: Give everyone a holistic view of your use case as it stands today.

- The problem your software/system solves (1-2 sentences)
- Use Case: high-level explainer (1-2 sentences)
- Why privacy matters for this specific use case (1-2 sentences)

- Start with the problem the use case solves
- Why is privacy relevant?

-->

---

![bg left:50% contain](images/edited/simple-architecture.png)

### Problem solved

- Teams must prove documentation meets quality and regulatory expectations
- Manual QM review is slow, inconsistent, and hard to scale
- AI agents can help — but introduce new privacy and data-flow risks

---

![bg left:50% contain](images/edited/simple-architecture.png)

### High-level architecture

- Human operator
- Next.js frontend
- FastAPI backend
- PostgreSQL database
- Orchestrator and agents

---

![bg left:50% contain](images/raw/simple-architecture.png)

### Privacy matters

- User accounts, roles, and session data (GDPR-relevant)
- Document content and audit trails may contain sensitive business data
- Egress to LLM providers and telemetry expand the trust boundary

---

<!-- class: sec2 dense -->
<!-- _class: sec2 divider -->

# Top privacy risks

- Egress traffic flow data to director and external models
- Application telemetry and workflow tracing
- Role-based access controls and GDPR compliance

---

# Up next: In-depth risk analysis

### Let's look at each privacy risk...

- its sensitive data
- why the risk matters
- the priority of the risk
- **mitigations** of the risk

---

<!-- class: sec31 -->
<!-- _class: sec31 divider section-intro -->

# Egress exposes privacy data

![](images/raw/risk-area-1.png)

---

![bg left:40% contain](images/raw/claude_1.png)
![bg left:40% contain](images/raw/claude_1.1.png)

### Sensitive data

- External model: transfer of potentially personal prompt/output data
  - **Data**:
    - Names, emails, stakeholder assignments, reviewer identifiers, document passages copied into prompts, generated summaries that may restate personal data
  - **Why Sensitive**:
    - Leaves the primary application context during model inference (current external-provider path)

---

![bg left:40% contain](images/raw/claude_2.jpg)

- Over-collection and -retention in model observability traces
  - **Data**:
    - provider, model_used, trace_id, correlation_id, latency/tokens,
      rich payload entries, prompt/output snapshots
  - **Why Sensitive**:
    - Enables reconstruction of _who_ triggered _what_ model action
      and _when_
    - can become re-identifiable when combined with audit tables

---

![bg left:40% contain](images/raw/claude_3.jpg)

- Model credentials and routing configuration
  - **Data**:
    - API keys, adapter routing flags, provider selection settings
  - **Why Sensitive**:
    - Compromise enables data exfiltration or unauthorized
      model usage

---

### Mitigation: Egress traffic flow data to orchestrator and models

---

<!-- _class: sec31 risk-diagram -->

### TODO: add happy mitigated image here

![](images/raw/risk-area-1.png)

---

<!-- class: sec32 -->
<!-- _class: sec32 divider section-intro -->

# Audit logs/reports and telemetry expose privacy data

![](images/raw/risk-area-2.png)

---

### Sensitive data

- Raw LLM prompt and output in _quality_observations.payload_
  - **Data**:
    - _payload.provider, payload.model_used, payload.llm_temperature_
    - _payload.llm_prompt_ (document text submitted by user, stakeholder names,
      reviewer assignments)
    - _payload.llm_output_ (generated compliance summaries restating personal
      context)

---

- - **Why Sensitive**:
    - Prompt context is assembled from user-submitted documents which routinely contain names, emails, project identifiers, and business-sensitive content
    - The output may restate or summarise that content
    - Both are persisted to a queryable table

---

- Audit event payload in audits_events.payload
  - **Data**:
    - payload.roles (user role at login), payload.remember_me flag, event-specific context fields, any free-form data inserted by orchestrator or Skills API callers
  - **Why Sensitive**:
    - Append-only and intended for long retention
    - No technical control prevents a caller from writing personal data into the payload column
    - Combined with actor_id (user email) it forms a rich personal profile

---

- OTEL (open telemetry) span attributes (exporter config)
  - **Data**:
    - HTTP path attribute (e.g., _/api/v1/documents/doc-abc123_ — document ID in path), user_agent, http.method, http.status_code, trace_id / span_id (linkable back to session)
  - **Why Sensitive**:
    - When TRACING_EXPORTER=otlp span data is sent to an external collector,
      - path values can embed document or session identifiers
      - user-agent can contribute to fingerprinting

---

- Frontend CSV export of prompt/output pairs
  - **Data**:
    - Exported observability _prompt_output_pairs.csv_ file containing prompt, output, source_component, trace_id, timestamps
  - **Why Sensitive**:
    - Created on the user's local filesystem;
    - outside system access controls, retention policy, and audit log;
    - may contain personal data from documents processed during the export
      window

---

### Mitigation: Application telemetry and workflow tracing

- Recommend following [OpenTelemetry best practices for handling sensitive data](https://opentelemetry.io/docs/security/handling-sensitive-data/)
- Follow the principle of data minimization, where possible.
- While it may not be optimal, try hashing user information and id and delete unhashed data.
- Consider using OTel's `redaction` processor between collection and export
- Apply similar principles to audit logs and reports

---

<!-- _class: sec32 risk-diagram -->

### TODO: add happy mitigated image here

![](images/raw/risk-area-2.png)

---

<!-- class: sec33 -->
<!-- _class: sec33 divider section-intro -->

# Misconfigured and stale controls give access to private data

![](images/raw/risk-area-3.png)

---

### Sensitive data

- User identity in session store
  - **Data**:
    - user_email (primary identifier), user_org, session session_id, expires_at, last_seen_at
  - **Why Sensitive**:
    - Directly identifies users and links their organisational role to access patterns and audit trails
    - Retained in PostgreSQL with no visible TTL-based purge policy

---

- Role assignments and permission scope
  - **Data**:
    - user_roles array per session row (e.g., ["qm_lead"], ["auditor"]), org isolation field user_org
  - **Why Sensitive**:
    - Reveals organisational responsibilities and access privileges
    - Can be used for social engineering or targeted attacks
    - GDPR data minimisation applies

---

- Bootstrap/MVP credentials in environment configuration
  - **Data**:
    - AUTH_MVP_EMAIL, AUTH_MVP_PASSWORD, AUTH_MVP_ROLES, AUTH_MVP_ORG (env vars)
    - SECRET_KEY (API key secret)
  - **Why sensitive**:
    - Compromise of bootstrap credentials gives attacker a fully provisioned account with configurable roles
    - SECRET_KEY grants service-client access to skills and observability

---

- Access decision and audit context not separately persisted
  - **Data**:
    - Which role accessed which route at what time
    - 403 access denials
    - service-client route usage with payload summary
  - **Why sensitive**:
    - Absence of access-decision audit log prevents retrospective investigation of data-access incidents
    - Required for GDPR Art. 30 record of processing activities and breach response

---

<!-- _class: sec33 mitigation-screens -->

### Mitigation: Role-based access controls and GDPR compliance

<div class="appendix-body appendix-duo">
  <img src="images/raw/Docquality_Admin-GovernanceControl.png" alt="Governance control" />
  <img src="images/raw/Docquality_Admin-RBAC.png" alt="Role-based access control" />
</div>

---

<!-- _class: sec33 risk-diagram -->

### TODO: add happy mitigated image here

![](images/raw/risk-area-3.png)

---

<!-- class: sec4 -->
<!-- _class: sec4 divider -->

# Recommendations

- Top priority going forward
- Why it matters
- Fit with company and risk strategy
- Value created

---

### Top priority going forward

- Continue to refine the automation and human interaction
- Focus on reducing manual work while meeting the standard
- Maintain and enhance the tool for furthre efficiency gains and respond to regulatory changes

---

### Why it matters

- Too much for any one human to analyze 1000 page document
- Reduces time to analyze document from weeks and months
- Flexibility to implement changes if directives change
- Reduces the need for expensive specialists to work on repetitive tasks
- Offers basic insights on what has changed, where changed, and policies affected

---

### Fit with company and risk strategy

- Complies with GDPR / BSI in Germany regulations for software rules
- Main regulatory points are addressed in workflow and prepares for external audits

---

### Value created

- Company is prepared for being audited by external agencies
- Cost savings from moving to internal models instead of using external services
- Maximize the human's impact for Human-in-the-loop by reducing toil and increasing time to focus on complex business cases

---

<!-- class: sec5 -->
<!-- _class: sec5 divider -->

# Questions & answers

- Audit scope and findings
- Risk prioritization and trade-offs
- Mitigations and open items

---

<!-- _class: sec5 -->

![bg right:45% contain](images/raw/claude_1.png)
![bg right:45% contain](images/raw/claude_1.1.png)
![bg right:45% contain](images/raw/claude_2.jpg)
![bg right:45% contain](images/raw/claude_3.jpg)

# Thank you

Ilona Brinkmeier · Wiebke Meyer · Carol Willing

**Privacy audit**
https://willingc.github.io/compliance-checker/

**Application**
https://github.com/IloBe/doc_quality_compliance_check

---

<!-- class: appendix -->
<!-- _class: appendix divider -->

# Appendix

- DocQuality Bridge workflow screens
- DocQuality Admin screens
- DocQuality Compliance standards screens

---

<!-- _class: appendix -->

### Bridge workflow — orchestration

<div class="appendix-body">
  <img src="images/raw/Docquality_Bridge-OrchestrationWorkflow.png" alt="Orchestration workflow" />
</div>

---

<!-- _class: appendix -->

### Bridge workflow — run steps 1–2

<div class="appendix-body appendix-duo">
  <img src="images/raw/Docquality_Bridge-WorkflowRun-1.png" alt="Workflow step 1" />
  <img src="images/raw/Docquality_Bridge-WorkflowRun-2.png" alt="Workflow step 2" />
</div>

---

<!-- _class: appendix -->

### Bridge workflow — run steps 4–4b

<div class="appendix-body appendix-duo">
  <img src="images/raw/Docquality_Bridge-WorkflowRun-4.png" alt="Workflow step 4" />
  <img src="images/raw/Docquality_Bridge-WorkflowRun-4b.png" alt="Workflow step 4 (detail)" />
</div>

---

<!-- _class: appendix -->

### Bridge workflow — run step 5

<div class="appendix-body">
  <img src="images/raw/Docquality_Bridge-WorkflowRun-5.png" alt="Workflow step 5" />
</div>

---

<!-- _class: appendix -->

### Admin — governance & access control

<div class="appendix-body appendix-duo">
  <img src="images/raw/Docquality_Admin-GovernanceControl.png" alt="Governance control" />
  <img src="images/raw/Docquality_Admin-RBAC.png" alt="Role-based access control" />
</div>

---

<!-- _class: appendix -->

### Compliance standards — mandatory & optional

<div class="appendix-body appendix-duo">
  <img src="images/raw/Docquality_ComplianceStandards-mandatory.png" alt="Mandatory compliance standards" />
  <img src="images/raw/Docquality_ComplianceStandards-optional.png" alt="Optional compliance standards" />
</div>
