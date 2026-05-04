# AI System Privacy Audit: Secrets, Tokens, and API Keys

System in scope: `doc_quality_compliance_check` — application secret management (`SECRET_KEY`, `DATABASE_URL`, `ANTHROPIC_API_KEY`, `PERPLEXITY_API_KEY`), session token lifecycle, password hashing, recovery token flow, bootstrap credential provisioning, and Docker Compose database credentials.

## 1. System Diagram

![](images/raw/compliance-checker-2026-05-02.png)

Secrets-relevant architecture facts used in this risk sheet:

- All sensitive configuration values are loaded from environment variables via Pydantic `BaseSettings` (`config.py`); a `.env` file is supported for local development (`.gitignore`d).
- **`SECRET_KEY`**: used as the HMAC secret for hashing session tokens and password recovery tokens. Default value is `"change-me-in-production"`. A `model_validator` in `Settings` raises `ValueError` if the default is detected in `environment == "production"` — but **not** in `staging`.
- **`DATABASE_URL`**: embeds database username and password in a connection string; default contains `CHANGE_ME` placeholder. No secrets manager integration is implemented.
- **`ANTHROPIC_API_KEY`** / **`PERPLEXITY_API_KEY`**: optional; empty string default means the LLM path is silently skipped when not configured. No key rotation, expiry check, or scoping mechanism is implemented.
- **Password storage**: PBKDF2-HMAC-SHA256 with 240 000 iterations, random 16-byte salt per password, stored as `pbkdf2_sha256$<iterations>$<salt_b64>$<hash_b64>` — a sound custom implementation, but not using a vetted library (e.g., `passlib`, `argon2-cffi`).
- **Session tokens**: generated with `secrets.token_urlsafe(48)`, HMAC-hashed (`SHA-256`) with `SECRET_KEY` before DB persistence; raw token sent to browser as HTTP-only cookie. Session table retains all rows (including revoked) indefinitely.
- **Recovery tokens**: generated with `secrets.token_urlsafe(48)`, hashed with `SECRET_KEY` + `SHA-256`, stored in `password_recovery_tokens` table; a **development debug flag** (`auth_recovery_debug_expose_token`) returns the raw plaintext token in the API response body — conditional on `environment == "development"`.
- **Docker Compose** (`docker-compose.yml`): PostgreSQL service uses `POSTGRES_PASSWORD: postgres` — a weak, hardcoded credential in a committed file.
- **Service-to-service API key** (`X-API-Key` / `Authorization: Bearer`): validated against `SECRET_KEY` value — the same secret serves dual duty as both the session-token HMAC key and the service-client API key.
- No secrets manager (Vault, AWS Secrets Manager, Azure Key Vault) or secret rotation mechanism is implemented.

## 2. Data Flow Analysis

| Data Flow | Source | Destination | Encrypted? | Logged? | Priority |
| --- | --- | --- | --- | --- | --- |
| `SECRET_KEY` loaded from env / `.env` file at startup | `.env` file or process environment | `Settings` singleton in memory → `session_auth.py`, `security.py`, `auth.py` | Process memory (no encryption at rest in env/file) | Not logged; used as HMAC material | High |
| `DATABASE_URL` (with embedded password) loaded at startup | `.env` file or process environment | SQLAlchemy engine connection pool | TLS in-transit to DB (depends on URL param); credential in plaintext in `DATABASE_URL` string | Not logged directly; may appear in debug/error traces if SQLAlchemy prints the full URL | High |
| `ANTHROPIC_API_KEY` / `PERPLEXITY_API_KEY` loaded at startup | `.env` file or process environment | `Settings` singleton → model adapter → external API call header | In-transit (HTTPS/TLS); key never returned in API responses | Not logged; could appear in provider SDK debug traces if `LOG_LEVEL=DEBUG` | High |
| Raw session cookie issued to browser | `create_server_session()` → `set_session_cookie()` | Browser HTTP-only cookie | In-transit (TLS); HTTP-only, `secure` flag enforced outside development | Raw token not persisted; only `session_token_hash` stored in DB | High |
| Session token hash stored in PostgreSQL | `session_auth.py` | `user_sessions` table — `session_token_hash` column | At-rest DB controls | Not separately logged; row retained after expiry/revocation | High |
| Service-client API key validated against `SECRET_KEY` | `X-API-Key` or `Authorization: Bearer` header | `require_api_auth()` in `security.py` — constant-time comparison via `hmac.compare_digest` | In-transit (HTTPS/TLS) | Not logged; single shared secret for all service clients — no per-client identity | High |
| Raw recovery token returned in API response (development debug mode) | `auth.py` → `POST /api/v1/auth/recovery/request` | Response body: `{ "debug_token": "<raw>", "reset_url": "..." }` | In-transit (TLS in dev only if `localhost`); but raw secret in plaintext HTTP body | Not logged to `audit_events`; only token hash and `expires_at` persisted | High |
| Recovery token hash stored in DB | `auth.py` | `password_recovery_tokens` table — `token_hash` column | At-rest DB controls | `auth.recovery.requested` event in `audit_events` | Medium |
| Bootstrap user credentials provisioned at startup | `AUTH_MVP_EMAIL`, `AUTH_MVP_PASSWORD` env vars → `AppUserORM` row | `app_users` table (password hashed with PBKDF2) | Env var source; hash stored at-rest | Startup event; plaintext password value is in env/config only | High |
| `POSTGRES_PASSWORD: postgres` in committed `docker-compose.yml` | Repository source control | Docker Compose → PostgreSQL container | Not encrypted (dev Docker network); hardcoded in version-controlled file | Committed to repository history | High |

### Corrected interpretation for data privacy and GDPR

- GDPR does not directly regulate secrets, but secret compromise directly enables data breaches, which must be reported under GDPR Art. 33. Every secret mismanagement risk in this sheet is therefore a potential GDPR breach enabler.
- The `SECRET_KEY` is a root-of-trust credential: if leaked, all existing session tokens can be forged (HMAC verification bypassed), all recovery tokens can be computed, and all service-client access can be replicated. This makes `SECRET_KEY` rotation the single highest-impact hardening action.
- The dual use of `SECRET_KEY` as both the session HMAC key and the service-client API key means a service-client credential leak also compromises session token integrity — and vice versa.
- The `DATABASE_URL` containing an embedded password is a known OWASP A02:2021 (Cryptographic Failures / Secret Exposure) pattern; if the URL appears in a stack trace, log file, or debug output it constitutes an unintended credential disclosure.

## 3. Sensitive Data

### Sensitive Data: `SECRET_KEY` — Root-of-Trust Session HMAC and Service API Secret

- **Category:** Cryptographic secret / system credential
- **Examples:** `SECRET_KEY=<64-char hex>` in `.env`; used in `_hash_session_token()`, `_hash_recovery_token()`, `require_api_auth()` — same value for all three purposes
- **Why Sensitive:** Compromise enables session token forgery (full account takeover for any user), recovery token prediction (password reset for any user), and unauthorised service-client access to `skills/*` and `observability/*` endpoints
- **Current Protection:** Env var / `.env` file (`.gitignore`d); production startup guard rejects default value; not logged
- **Risk (or Harm) if Exposed:** Full system compromise; GDPR Art. 33 breach notification required; all user sessions must be invalidated; all recovery tokens must be rotated

### Sensitive Data: `DATABASE_URL` Embedding Plaintext Password

- **Category:** Infrastructure credential / connection string
- **Examples:** `postgresql+psycopg2://dbuser:CHANGE_ME@localhost:5432/doc_quality`; default in `config.py` with `nosec B105` suppression; also in `.env.example`
- **Why Sensitive:** Database credential grants direct read/write access to all personal data: `app_users`, `user_sessions`, `audit_events`, `quality_observations`, `bridge_human_reviews` — the full system data estate
- **Current Protection:** `.env` file excluded from version control; `CHANGE_ME` placeholder in default; no rotation mechanism
- **Risk (or Harm) if Exposed:** Direct database access bypassing all RBAC; bulk personal data exfiltration; modification or deletion of audit trail; GDPR Art. 33 major breach

### Sensitive Data: `ANTHROPIC_API_KEY` / `PERPLEXITY_API_KEY` — External Provider Credentials

- **Category:** Third-party API credentials
- **Examples:** `ANTHROPIC_API_KEY=sk-ant-...`; `PERPLEXITY_API_KEY=pplx-...`
- **Why Sensitive:** Compromise enables unauthorised model usage billed to the organisation; more critically, the API key is sent in the `Authorization` header of every model call — if an attacker can replay this key they can submit arbitrary prompts to the provider's API, potentially exfiltrating system prompts or bypassing rate limits
- **Current Protection:** Env var / `.env` file; empty string default (LLM path disabled if not set); not logged at `INFO` level
- **Risk (or Harm) if Exposed:** Unauthorised model API usage and cost; prompt injection via replayed API calls; no per-call or per-context scoping of the key

### Sensitive Data: `POSTGRES_PASSWORD: postgres` Hardcoded in `docker-compose.yml`

- **Category:** Infrastructure credential committed to version control
- **Examples:** `POSTGRES_PASSWORD: postgres` on line 12 of `docker-compose.yml`
- **Why Sensitive:** Hardcoded weak credential is committed to the repository; any person with repository read access has the development database password; if the same password is accidentally reused in staging or production (common operational error), database access is trivially compromised
- **Current Protection:** Intended for local development only; Docker network not exposed beyond `localhost` in default config
- **Risk (or Harm) if Exposed:** Credential reuse in non-development environments; repository access = database credential; OWASP A07:2021 (Identification and Authentication Failures)

### Sensitive Data: Recovery Token Debug Exposure (`auth_recovery_debug_expose_token`)

- **Category:** Authentication token exposed in API response body
- **Examples:** `{ "debug_token": "<raw_token_urlsafe_48>", "reset_url": "http://localhost:3000/reset-access?token=..." }` returned by `POST /api/v1/auth/recovery/request` when `AUTH_RECOVERY_DEBUG_EXPOSE_TOKEN=true`
- **Why Sensitive:** A raw, single-use password-reset token returned in plaintext in an HTTP response body is a high-value target; if the flag is accidentally enabled outside a development sandbox (e.g., in a shared staging environment), any observer of the response (log aggregator, proxy, developer tooling) captures a live reset credential
- **Current Protection:** Conditional on `environment == "development" AND auth_recovery_debug_expose_token == True`; default is `False`
- **Risk (or Harm) if Exposed:** Account takeover via captured recovery token; GDPR Art. 33 breach notification required if a real user's account is affected

## 4. Privacy Risks

### Risk 1: `SECRET_KEY` serves dual purpose — session HMAC and service-client API key

- **Priority:** High
- **Risk Category:** Cryptographic key management and separation of concerns
- **GDPR Reference:** Art. 32 — appropriate technical measures; Art. 33 — breach notification (key compromise = system-wide breach)
- **Potential Harm/Impact:** A single leaked `SECRET_KEY` value simultaneously: (a) allows session token forgery for any user account, (b) allows recovery token prediction enabling password reset for any user, (c) grants service-client access to `skills/*` and `observability/*` endpoints. There is no way to rotate service-client credentials without also invalidating all active user sessions — no surgical incident response is possible
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Introduce separate secrets: `SESSION_HMAC_KEY` (session token HMAC), `RECOVERY_HMAC_KEY` (recovery token HMAC), `SERVICE_API_KEY` (service-client authentication) — three independent values, independently rotatable.
  - Add a `SECRET_KEY_VERSION` or key ID mechanism so that token hashes can identify which key generation they were produced with, enabling zero-downtime rotation.
  - Document a key rotation runbook: how to rotate each key independently, what sessions are invalidated, and how service clients are notified.

### Risk 2: No startup guard prevents use of default `SECRET_KEY` in staging

- **Priority:** High
- **Risk Category:** Configuration hardening and environment parity
- **GDPR Reference:** Art. 32 — security of processing; Art. 25 — data protection by design
- **Potential Harm/Impact:** The `model_validator` in `Settings` only rejects the default `"change-me-in-production"` when `environment == "production"`. A staging environment with `environment = "staging"` and the default key is fully operational — all sessions are signed with a publicly known key value, making every user session trivially forgeable
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Extend the `validate_security_defaults` validator to also reject default secret values when `environment in ("staging", "production")`.
  - Add the same guard to `AUTH_MVP_PASSWORD` (`CHANGE_ME_BEFORE_USE`) and `DATABASE_URL` password (`CHANGE_ME`) for non-development environments.
  - Consider a CI pre-deployment check that reads the environment's `SECRET_KEY` and fails the pipeline if it matches any known default.

### Risk 3: `DATABASE_URL` embeds plaintext database password in a config string

- **Priority:** High
- **Risk Category:** Credential exposure in configuration — OWASP A02:2021
- **GDPR Reference:** Art. 32 — appropriate technical measures; Art. 33 — breach enabler
- **Potential Harm/Impact:** SQLAlchemy may include the full `DATABASE_URL` in exception messages and debug output; if `LOG_LEVEL=DEBUG` or an unhandled exception is logged, the database password appears in the log stream and potentially in the log aggregator (see Risk Sheet: Structured Logging, Risk 3); the placeholder `CHANGE_ME` in `config.py` and `.env.example` risks being reused directly
- **Ability to Implement Control:** Medium
- **Recommended controls:**
  - Use a secrets manager (Vault, AWS Secrets Manager, Azure Key Vault) to inject `DATABASE_URL` at runtime rather than sourcing from a `.env` file.
  - As an intermediate measure, configure SQLAlchemy with `hide_parameters=True` on the engine to prevent credential exposure in exception output.
  - Add a startup validator: if `DATABASE_URL` contains the literal string `CHANGE_ME`, reject startup with a clear error in all non-development environments.
  - Use PostgreSQL's `PGPASSFILE` (`.pgpass`) or `PGPASSWORD` environment separation instead of embedding the password in the URL.

### Risk 4: Hardcoded `POSTGRES_PASSWORD: postgres` committed to version control

- **Priority:** High
- **Risk Category:** Credential committed to source control — OWASP A07:2021
- **GDPR Reference:** Art. 32 — security of processing
- **Potential Harm/Impact:** Any developer or automated system (CI/CD, GitHub Actions, Dependabot) with repository read access has the development database credential; if this password is reused — even accidentally — in a non-development environment, the database is trivially accessible; the credential is also in repository history and cannot be removed without a full `git filter-branch` / BFG rewrite
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Replace the hardcoded value with an environment variable reference in `docker-compose.yml`: `POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-postgres}` — keeping `postgres` as a local-only default that is never committed as a required credential.
  - Add a `docker-compose.override.yml` or `.env.docker` file (both `.gitignore`d) for developer-specific values.
  - Add a pre-commit hook or CI secret-scanning step (e.g., `trufflehog`, `gitleaks`) to prevent future hardcoded secrets from reaching the repository.

### Risk 5: Recovery token debug exposure flag — risk of misuse outside development sandbox

- **Priority:** High
- **Risk Category:** Debug backdoor in authentication flow
- **GDPR Reference:** Art. 32 — security of processing; Art. 33 — breach notification if exploited
- **Potential Harm/Impact:** `AUTH_RECOVERY_DEBUG_EXPOSE_TOKEN=true` returns a live, single-use password-reset token in the HTTP response body. If set on a shared staging environment (shared with QA engineers, product reviewers, or frontend developers), any developer with access to browser developer tools or a shared proxy captures a live credential for any user who requests a password reset during that period
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Add an additional guard: refuse to activate this flag if more than one user record exists in `app_users` (i.e., only safe in single-user bootstrap scenarios).
  - Add a startup warning log entry when this flag is enabled: `logger.warning("SECURITY_WARNING: auth_recovery_debug_expose_token is enabled — DO NOT USE IN SHARED ENVIRONMENTS")`.
  - Remove the flag from `.env.example` entirely so it is never accidentally copied into a shared environment configuration.
  - Consider replacing this debug pattern with a test-only token endpoint that is compiled out of production builds.

### Risk 6: No API key scoping, rotation, or per-client identity for service-to-service authentication

- **Priority:** Medium
- **Risk Category:** Service credential lifecycle and least-privilege
- **GDPR Reference:** Art. 25 — data protection by design; Art. 32 — security of processing
- **Potential Harm/Impact:** A single `SECRET_KEY`-based API key authenticates all service clients (CrewAI orchestrator, automation tools, test harnesses) with the `service` role. There is no per-client identity, no expiry, no revocation mechanism, and no audit trail distinguishing which service client made a given request. If the key is shared with a third-party integration or leaked from one service, all service clients are compromised simultaneously
- **Ability to Implement Control:** Medium
- **Recommended controls:**
  - Issue per-client API keys (e.g., `ORCHESTRATOR_API_KEY`, `AUTOMATION_API_KEY`) each stored separately; validate against a key registry rather than a single shared secret.
  - Assign each service client a unique `actor_id` (e.g., `orchestrator-v1`, `ci-automation`) in the `require_api_auth` response so that service-client requests are attributable in `audit_events`.
  - Add key expiry and a rotation schedule (e.g., 90-day rotation enforced via startup validator).

### Risk 7: PBKDF2 password hashing is a custom implementation without a vetted library

- **Priority:** Medium
- **Risk Category:** Cryptographic implementation risk
- **GDPR Reference:** Art. 32 — appropriate technical measures
- **Potential Harm/Impact:** The custom PBKDF2-HMAC-SHA256 implementation in `passwords.py` is functionally correct (240 000 iterations, 16-byte random salt, HMAC-based constant-time comparison), but custom cryptographic code carries implementation risk: iteration count is hardcoded (not configurable for future increases), there is no algorithm migration path if PBKDF2 needs to be replaced with Argon2id, and the format string `pbkdf2_sha256$...` mimics Django's format but is not interoperable with standard tools
- **Ability to Implement Control:** High
- **Recommended controls:**
  - Replace the custom implementation with `passlib` (with `argon2` backend) or `argon2-cffi` directly — both are vetted, actively maintained libraries with built-in migration support.
  - If PBKDF2 must be retained, make `iterations` a configurable setting (`PASSWORD_HASH_ITERATIONS`) and add a startup validator that rejects values below 260 000 (the 2023 OWASP minimum).
  - Add a password rehash-on-login path: when a user successfully logs in and their stored hash uses outdated parameters, silently re-hash with current parameters.

## 5. Cross-Sheet Consistency

| Control Area | Related Risk Sheet | Alignment Required |
| --- | --- | --- |
| `SECRET_KEY` rotation impacts all active sessions | Risk Sheet 2 (RBAC, Risk 2) | Session table purge (recommended in sheet 2) must be coordinated with key rotation — revoke all sessions before rotating `SECRET_KEY` |
| `DATABASE_URL` password in log traces | Risk Sheet 3b (Structured Logging, Risk 2) | DEBUG log guard must cover SQLAlchemy engine initialisation path to prevent URL (with password) from appearing in debug output |
| `ANTHROPIC_API_KEY` / `PERPLEXITY_API_KEY` in provider SDK debug output | Risk Sheet 3b (Structured Logging, Risk 2) | Provider SDK log suppression must cover API key header values |
| Service-client API key — single shared secret | Risk Sheet 2 (RBAC, Risk 5) | Per-client API keys enable attribution of service-client access to `observability/*` (Risk 5 sheet 2) — dependent fix |
| Recovery token debug flag | Risk Sheet 3b (Structured Logging) | If `debug_token` is ever accidentally logged (e.g., by a request-body logger middleware), it constitutes a credential exposure in the log stream — body logging must never capture auth endpoint responses |
| `POSTGRES_PASSWORD` in docker-compose | Risk Sheet 1 (Model Providers, Risk 1) | On-prem model migration requires a private network DB configuration — the same docker-compose pattern must not be used for production DB connectivity |
