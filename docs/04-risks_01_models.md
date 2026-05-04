# Risk level 1: Egress to AI orchestrator and models

**AI System Privacy Audit: Documentation Quality Compliance Checker**

!!! note Priority 1

    Egress to CrewAI orchestrator and AI Models (including
    Perplexity, Anthropic, and Others)

## 1. System Diagram

![](images/raw/compliance-checker-2026-05-02.png)

## 2. Data Flow Analysis

| Data Flow    | Source  | Destination | Encrypted? | Logged? | Priority |
| ------------ | ------- | ----------- | ---------- | ------- | -------- |
| User input   | Web app | LLM API     | In-transit | Yes     | High     |
| Model output | LLM API | Database    | No         | 50%     | Medium   |

## 3. Sensitive Data

### Sensitive Data Name: [external model]

- **Category:**
- **Examples:**
- **Why Sensitive:**
- **Current Protection:**
- **Risk (or Harm) if Exposed:**

## 4. Privacy Risks

### Risk 1: Egress to Orchestrator and AI Models

- **Priority:** [High]
- **Risk Category:** [model (please describe!)]
- **Potential Harm/Impact:** What happens if this risk isn't addressed?
- **Ability to Implement Control:** [Low/Medium/High]
- **Recommended controls:**
  - usage of local model
  - evaluate models via model cards
  - quality management needs to be defined
  - ....

---

## Additional information from the repo

Subprocessor egress — Anthropic and Perplexity

### Perplexity (`src/doc_quality/services/research_service.py`)

**When:** `PERPLEXITY_API_KEY` is non-empty.

**Prompt content:** Built from `ResearchRequest`: `domain`, `description`, `target_market`, and optional `custom_query` (or default research question) via `prompts/research_prompt_v1.txt`.

**When key is empty:** Static fallback; **no** HTTP call to Perplexity (still builds `query` string for `ResearchResult`).

### Research API persistence (`src/doc_quality/api/routes/research.py`)

When `result.provider == "perplexity"`, the API calls `create_quality_observation` with `payload` containing:

- `llm_prompt` (full built prompt),
- `llm_output` (full model answer),
- provider metadata and citation counts.

These are stored in **`quality_observations.payload`** (`src/doc_quality/services/quality_service.py`). This duplicates potentially sensitive user text **on disk** in addition to sending it to Perplexity.

### Anthropic — risk template AI suggest (`src/doc_quality/api/routes/risk_templates.py`)

**When:** `ANTHROPIC_API_KEY` set; route `ai-suggest`.

**Sent to Anthropic:** System prompt (FMEA vs RMF), optional `request.context` (product/system context), `partial_row` as JSON, and instructions. **Not** full documents unless `context` or row fields contain them.

### Anthropic — document analysis agent (`src/doc_quality/agents/doc_check_agent.py`)

**Implementation:** Sends up to **2000 characters** of document `content` plus `document_type` and rule-based `issues` in the user message.

**Wiring:** `DocumentCheckAgent` is **not referenced** elsewhere in the Python tree (only defined in this file). The live **`analyze_document`** path uses **`document_analyzer.analyze_document`** only — **rule-based, no LLM** in current API flows. Treat as dead integration risk if something imports it later.

### Compliance agent (`src/doc_quality/agents/compliance_agent.py`)

Creates an Anthropic client when a key exists but **`check_compliance`** delegates to **`compliance_checker`** only (no Anthropic messages observed in `compliance_checker.py`).

### Orchestrator service (`services/orchestrator/`)

Separate process; `AnthropicAdapter` is scaffold-style. Boundary review: whatever workflows you enable may forward document fragments to backends — trace per workflow when operating this service.

### Subprocessor summary

| Vendor | Trigger | Typical data categories |
| --- | --- | --- |
| **Perplexity** | `/api/v1/research/regulations` with API key | Product domain, description, markets, optional free-text query → **also persisted** in DB as prompt+answer. |
| **Anthropic** | Risk template `ai-suggest` | Free-text context + partial row JSON. |
| **Anthropic** | `DocumentCheckAgent` | First 2k chars of document + issues — **not** used by default document routes today. |
