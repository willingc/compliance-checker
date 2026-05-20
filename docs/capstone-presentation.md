---
marp: true
theme: gaia
---

# AI Privacy Capstone

## Documentation Quality Compliance Checker

### Ilona Brinkmeier, Wiebke, Carol Willing

May 2026

<!--
Title slide speaker notes
-->

---

*1.0*

# Audit subject and use case

Subject: https://github.com/IloBe/doc_quality_compliance_check
Detailed privacy audit: https://willingc.github.io/compliance-checker/

## Problem solved

Insert brief description of the problem solved

## Privacy matters

Insert brief list of why privacy matters

<!--
## 1. Introduce your Use Case (~2 min)

Goal: Give everyone a holistic view of your use case as it stands today.

---

Include:

- Use Case: high-level explainer (1-2 sentences)
- The problem your software/system solves (1-2 sentences)
- Why privacy matters for this specific use case (1-2 sentences)

Tips:

- Start with the problem the use case solves
- Why is privacy relevant?
- Keep it under 2 minutes

-->

---
*1.1*
**High-level architecture**

- Human operator
- NextJS frontend
- FastAPI backend
- Postgres database
- Orchestrator and agents

![bg right:60% width:720](images/raw/simple-architecture.png)

---

*1.2*

### Real-world use case

Problem solved

---

*1.3*

### Privacy matters

---

*2.0*

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

*2.1*
**Risk Area 1**

### Egress traffic flow data to orchestrator and models

Sensitive data:

- External model: transfer of potentially personal prompt/output data
  // -- write down sensitivity reason or leave it to tell?
- Over-collection and -retention in model observability traces
- Model credentials and routing configuration

---

*2.2*
**Risk Area 2**

### Application telemetry and workflow tracing

Sensitive data:

- Raw LLM promp and output in quality_observations.payload
- Audit event payload in audits_events.payload
- OTEL (open telemetry) span attributes (exporter config)
- Frontend CSV export of prompt/output pairs

---

*2.3*
**Risk Area 3**

### Role-based access controls and GDPR compliance

Sensitive data:

- User identity in session store
- Role assignments and permission scope
- Bootstrap/MVP credentials in environment configuration
- Access decision and audit context not separately persisted

---

*3.0*

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

*3.1*
**Mitigating Risk 1**

### Egress traffic flow data to orchestrator and models

---

*3.2*
**Mitigating Risk 2**

### Application telemetry and workflow tracing

---

*3.3*
**Mitigating Risk 3**

### Role-based access controls and GDPR compliance

---

*4.0*

# Recommendation

## Top priority

- Insert what it is
- Insert why it is important
- Insert why it matters to company and risk strategy
- Insert what value does it create

<!--
## 4. Top Next Priority and Why (~3 min)

Goal: Practice executive communication: Convince your boss (or your boss's boss :) ).

---

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

*5.0*

# Questions and answers

Thank you!

Ilona, Wiebke, Carol
