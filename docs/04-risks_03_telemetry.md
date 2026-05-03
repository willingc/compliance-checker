# AI System Privacy Audit: [System Name]

_DRAFT - Needs to be edited_

## 1. System Diagram

![](images/raw/compliance-checker-2026-05-02.png)

## 2. Data Flow Analysis

| Data Flow    | Source  | Destination | Encrypted? | Logged? | Priority |
| ------------ | ------- | ----------- | ---------- | ------- | -------- |
| User input   | Web app | LLM API     | In-transit | Yes     | High     |
| Model output | LLM API | Database    | No         | 50%     | Medium   |

## 3. Sensitive Data

### Sensitive Data Name: [Brief description]

- **Category:**
- **Examples:**
- **Why Sensitive:**
- **Current Protection:**
- **Risk (or Harm) if Exposed:**

## 4. Privacy Risks

### Risk 1...: [Brief description]

- **Priority:** [Low/Medium/High]
- **Risk Category:** [model/input/additional data/other (please describe!)]
- **Potential Harm/Impact:** What happens if this risk isn't addressed?
- **Ability to Implement Control:** [Low/Medium/High]
- **Recommended controls:**
  - (your first guess, ok to not know and leave blank)
