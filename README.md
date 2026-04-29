# compliance-checker
Privacy audit for [doc quality compliance checker](https://github.com/IloBe/doc_quality_compliance_check)

This repo holds information related to a privacy audit.

## Project for audit

https://github.com/IloBe/doc_quality_compliance_check by Ilona Brinkmeier

## Use cases (week 1)

- use case 1: validate basic auth privacy controls (session cookie + no plain credentials persistence)
- use case 2: validate password recovery privacy safety (recovery token TTL + single-use)
- use case 3: verify that PII-adjacent endpoints are role-protected (RBAC protection of PII access)
- use case 4: verify user-supplied text is sanitized before being persistence or used (input sanitization of potential PII payload)
- use case 5: verify machine client scope and privacy auditability (service-client boundary + traceability consistency)

## Data flow diagrams (week 1)

