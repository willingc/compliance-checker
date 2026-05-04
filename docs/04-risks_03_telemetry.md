# Risk level 3: Telemetry

**AI System Privacy Audit: Documentation Quality Compliance Checker**

!!! note Priority 3

    Telemetry for evaluation user workflow tracking

## 1. System Diagram

![](images/raw/compliance-checker-2026-05-02.png)

## 2. Data Flow Analysis

| Data Flow    | Source  | Destination | Encrypted? | Logged? | Priority |
| ------------ | ------- | ----------- | ---------- | ------- | -------- |
| User input   | Web app | LLM API     | In-transit | Yes     | High     |
| Model output | LLM API | Database    | No         | 50%     | Medium   |

## 3. Sensitive Data

### Sensitive Data Name: Telemetry information

- **Category:**
- **Examples:**
- **Why Sensitive:**
- **Current Protection:**
- **Risk (or Harm) if Exposed:**

## 4. Privacy Risks

### Risk 3: Telemetry for evaluation workflow tracking

- **Priority:** [Medium]
- **Risk Category:** [model/input/additional data/other (please describe!)]
- **Potential Harm/Impact:** What happens if this risk isn't addressed?
- **Ability to Implement Control:** [Low/Medium/High]
- **Recommended controls:**
  - (your first guess, ok to not know and leave blank)

---

## Additional information from the repo

### OpenTelemetry and logs (request workflow)

- **Tracing:** `configure_observability` in `observability.py` registers a `TracerProvider` when `tracing_enabled` is true (OTLP or console exporter, sampling via `tracing_sampling_ratio`). **HTTP middleware** in `main.py` starts a span per API request and attaches standard HTTP attributes.
- **Logs:** Each request logs **`http_request`** with `method`, `path`, `status`, `duration_ms`, and optional **`trace_id`** when a valid span context exists.

These mechanisms trace **API request execution**, not a separate end-user “clickstream” product analytics layer.

### Tests as examples of evaluation workflows

`tests/test_observability_api.py` demonstrates:

- Posting observations with **`evaluation_dataset`** / **`evaluation_metric`** on non-evaluation aspects (still counted toward **`evaluation_observations`** in the summary when `evaluation_dataset` is set).
- Posting **`aspect: "evaluation"`** with LLM fields in **`payload`** and retrieving them via **`/llm-traces`**.
- Populating **workflow component breakdown** with multiple **`source_component`** values.

### Configuration touchpoints

- **Service name:** `TELEMETRY_SERVICE_NAME` (e.g. in `.env.example`) feeds OpenTelemetry **`service.name`**.
- **Tracing/metrics flags and OTLP endpoint:** see `Settings` in `src/doc_quality/core/config.py` (tracing exporter, `tracing_otlp_endpoint`, `metrics_enabled`, etc.).

### Related documentation

- **`OBSERVABILITY_LOGGING_README.md`** — deeper operational logging and observability guide.
- **`README.md`** — Admin Observability overview and RBAC for `/api/v1/observability/*`.
