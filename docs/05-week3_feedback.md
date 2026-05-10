# Feedback on risks

!!! note "Week 3"

    Feedback from kjam on risk threat modeling.

    Action plan: Responses are in-line or marked as TODO

## Kudos

- Your data flow + related flow risk table (/04-risks_01_models.md) is looking really solid! I notice you are able to identify the sensitive flow, prioritize based on your risk analysis and already have great suggestions for prioritization and mitigation. Keep it up, looking amazing!

- You already identified an upcoming challenge. How do we observe and monitor our systems to guarantee accuracy and support security without overexposing privacy? This is a tension in most production systems and one we will come back to several times.

## Notes and Tips

- Although information security controls (like user management, passwords, secrets, encryption at rest and access controls) are vital to protect all data, but we tend not to mention them very much when doing privacy audits unless something has been compromised. Why? Although they may be necessary components, let's say of a house. Maybe privacy is kind of like the roof. Although we need the walls in order to build a roof, the walls do not actually "make" a good roof. Information security controls are a prerequisite, but not a privacy control in-and-of-itself (which you note, but I am just restating for clarity).

- Noted.
- **Given this feedback, it may be best to focus almost entirely on [models risk](04-risks_01_models.md) for the remainder of the project.**
- Of the remaining risk sections, we can aim for good software development practices and focus this privacy project mostly on models risk. One area in [access controls](04-risks_01_models.m) that we can address while tackling models risk is making sure all api requests are authenticated and authorized.

- Since you are operating under GDPR principles, it might help to also review what data is required for your operating proceedures, especially for regulatory audit purposes or for legitimate interest (https://gdpr-info.eu/art-6-gdpr/ (see b, c, e, f)). For example, if you need to track actor_id as a part of your business operations, so long as you do so responsibly, you are usually OK to do so. :)

    - **TODO: REVIEW**
    - Next action: review GDPR docs. Determine what data needs to be tracked.

## Questions

### 1. For your #2 prioritized risk in the 04-risks_01_models.md table, what exactly do you mean by: Data schema for validation?

Use pydantic/FastAPI library to validate the data meets the required schema.

### 2. Risks #3 and Risks #4

Risk #3 in the area below the data flows is about hallucination in compliance decisions and Risk #4 is about robustness of those decisions.

Does this have a privacy impact?

**Risk 3 (Hallucination)**

- Privacy impact: Low
- Hallucination seems to have more of a business risk than a privacy risk.
- Hallucinated results could have some private data returned in the reponse; though,
  that risk would be addressed when mitigating response / output privacy.

**Risk 4 (Robustness of compliance decisions, Model/version drift)**

- Privacy impact: Low
- Model/version drift falls more in the category of application reliability and robustness
- Measuring drift makes sense from a business standpoint but is unlikely to add additional privacy risk.
- It would make business sense to revisit highest priority privacy risks at the same time the model drift is checked. Perhaps quarterly or twice a year. Therefore, building in a planned cadence for addressing both on a regular basis.
