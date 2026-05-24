---
marp: true
theme: rose-pine
footer: "AI Privacy Capstone - May 2026"
paginate: true
---

<!-- _class: title -->

![bg left:20% cover](images/pexels-pixabay-207580.jpg)

# Documentation Quality Compliance Checker

Ilona Brinkmeier · Wiebke Meyer · Carol Willing

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

# Privacy risks

- Top privacy risks
- Risk areas in depth
- Prioritization and trade-offs

---

_2.0_

# Top privacy risks

1. Egress traffic flow data to orchestrator and models
2. Application telemetry and workflow tracing
3. Role-based access controls and GDPR compliance

<!--
## 2. Top Risks Identified (~4 min)

Goal: Demonstrate your ability to identify and prioritize real world AI privacy risk.

---

Include:

- Top 3 risks you identified and prioritized
- Why each risk matters (harms, severity, business and/or customer impact, compliance)
- How you prioritized risk, including one example of a top risk you deprioritized and why

Tips:

- Focus on impact and control, why was this important and how/why did you prioritize?
- Connect to real-world consequences (fines, user harm, reputation)
- Keep it AI-centric if possible (that's what this class is about!)

-->

---

_2.1_ **Risk Area 1**

### Egress traffic flow data to orchestrator and models

![bg left:40% contain](images/raw/claude_1.png)
![bg left:40% contain](images/raw/claude_1.1.png)

Sensitive data:

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

![](images/raw/risk-area-1.png)

---

_2.2_ **Risk Area 2**

### Application telemetry and workflow tracing

Sensitive data:

- Raw LLM promp and output in _quality_observations.payload_
  - **Data**:
    - _payload.provider, payload.model_used, payload.llm_temperature_
    - _payload.llm_prompt_ (document text submitted by user, stakeholder names,
      reviewer assignments)
    - _payload.llm_output_ (generated compliance summaries restating personal
      context)

---

- - **Why Sensitive**:
    - Prompt context is assembled from user-submitted documents which routinely
      contain names, emails, project identifiers, and business-sensitive content
    - The output may restate or summarise that content
    - Both are persisted to a queryable table

---

- Audit event payload in audits_events.payload
  - **Data**:
    - payload.roles (user role at login), payload.remember_me flag,
      event-specific context fields, any free-form data inserted by orchestrator or Skills API callers
  - **Why Sensitive**:
    - Append-only and intended for long retention
    - No technical control prevents a caller from writing personal data into the payload column
    - Combined with actor_id (user email) it forms a rich personal profile

---

**Risk Area 2**

- OTEL (open telemetry) span attributes (exporter config)
  - **Data**:
    - HTTP path attribute (e.g., _/api/v1/documents/doc-abc123_ — document ID in path), user_agent, http.method, http.status_code, trace_id / span_id (linkable back to session)
  - **Why Sensitive**:
    - When TRACING_EXPORTER=otlp span data is sent to an external collector,
      - path values can embed document or session identifiers
      - user-agent can contribute to fingerprinting

---

**Risk Area 2**

- Frontend CSV export of prompt/output pairs
  - **Data**:
    - Exported observability _prompt_output_pairs.csv_ file containing prompt, output, source_component, trace_id, timestamps
  - **Why Sensitive**:
    - Created on the user's local filesystem;
    - outside system access controls, retention policy, and audit log;
    - may contain personal data from documents processed during the export
      window

---

_2.3_ **Risk Area 3**

### Role-based access controls and GDPR compliance

Sensitive data:

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

<!-- class: sec3 -->
<!-- _class: sec3 divider -->

# Mitigations & controls

- Mitigation per top risk
- Technical approach and trade-offs
- Outcomes and learnings

---

_3.0_

# Mitigations and controls

- Mitigation implemented
- Technical details
- Selected approach vs. alternatives
- Outcome and learnings

<!--
## 3. How You Addressed Them (Mitigations + Controls) (~4 min)

Goal: Demonstrate your technical competence and decision-making.

---

Include:

- For each top risk: what mitigation you implemented
- Technical details: tools, frameworks, specific implementations
- Why you chose these approaches over alternatives
- What worked, what didn't, what you'd change

Tips:

- Be specific — mention concrete tools, parameters, configurations
- Explain your trade-offs — nothing is perfect
- Show you tested your solutions (red teaming, evaluations, etc.)

-->

---

_3.1_ **Mitigating Risk 1**

### Egress traffic flow data to orchestrator and models

---

_3.2_ **Mitigating Risk 2**

### Application telemetry and workflow tracing

---

_3.3_ **Mitigating Risk 3**

### Role-based access controls and GDPR compliance

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

- Meet the standard directives and too much to do manually
- Maintain tool for efficiency

<!--
## 4. Top Next Priority and Why (~3 min)

Goal: Practice executive communication: Convince your boss (or your boss's boss :) ).

Include:

- What's your #1 priority going forward?
- Why is it important? (ROI, risk reduction, user impact, compliance, security, etc...)
- How does it fit into the broader company and risk strategy?
- What value does it create?

Tips:

- This is management pitch practice. Think about how a company around this might evaluate ROI and risk.
- Connect to business/operational outcomes, not just technical improvements
- Show you understand prioritization and budgets. not everything is a fire

-->

---

### Why it matters

- Insert why it is important (ROI, risk reduction, user impact, compliance, security)
- Too much for any one human to analyze 1000 page document
- Time for weeks and months for a document
- Changes if directives change
- Expensive specialists
- Basic insights, what has changed, where changed, policies affected

---

### Fit with company and risk strategy

- GDPR / BSI in Germany regulations for software rules
- Main points are addressed for external audits

---

### Value created

- Being audit 

---

<!-- class: sec5 -->
<!-- _class: sec5 divider -->

# Questions & answers

- Audit scope and findings
- Risk prioritization and trade-offs
- Mitigations and open items

---

<!-- _class: sec5 -->

![bg right:30%](images/pexels-ozum-afsar-1430297-7908609.jpg)

# Thank you

Ilona Brinkmeier · Wiebke Meyer · Carol Willing

**Privacy audit**
https://willingc.github.io/compliance-checker/
