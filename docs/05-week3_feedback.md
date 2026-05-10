# Feedback on risks

!!! note "Week 3"

    Feedback on risk threat modeling.

## Kudos

- Your data flow + related flow risk table (/04-risks_01_models.md) is looking really solid! I notice you are able to identify the sensitive flow, prioritize based on your risk analysis and already have great suggestions for prioritization and mitigation. Keep it up, looking amazing!

- You already identified an upcoming challenge. How do we observe and monitor our systems to guarantee accuracy and support security without overexposing privacy? This is a tension in most production systems and one we will come back to several times.

## Notes and Tips

- Although information security controls (like user management, passwords, secrets, encryption at rest and access controls) are vital to protect all data, but we tend not to mention them very much when doing privacy audits unless something has been compromised. Why? Although they may be necessary components, let's say of a house. Maybe privacy is kind of like the roof. Although we need the walls in order to build a roof, the walls do not actually "make" a good roof. Information security controls are a prerequisite, but not a privacy control in-and-of-itself (which you note, but I am just restating for clarity).

- Since you are operating under GDPR principles, it might help to also review what data is required for your operating proceedures, especially for regulatory audit purposes or for legitimate interest (https://gdpr-info.eu/art-6-gdpr/ (see b, c, e, f)). For example, if you need to track actor_id as a part of your business operations, so long as you do so responsibly, you are usually OK to do so. :)

## Questions

1. For your #2 prioritized risk in the 04-risks_01_models.md table, what exactly do you mean by: Data schema for validation?

   > Use pydantic/FastAPI library to validate the data meets the required schema

2. Risk #3 in the area below the data flows is about hallucination in compliance decisions and Risk #4 is about robustness of those decisions.
- Does this have a privacy impact?
- Can you explain it a bit more in depth? (In my mind maybe this falls under reliability, security and robustness, but maybe I am missing something.)
