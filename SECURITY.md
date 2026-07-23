# SECURITY.md — PopUp-LA

Secure-coding rules and security guardrails for PopUp-LA. Sources of truth:
[REQUIREMENTS.md](REQUIREMENTS.md) (§19–22 AUTHZ/SEC/AUDIT/PRIV are binding),
[ARCHITECTURE.md](ARCHITECTURE.md), and [DESIGN.md](DESIGN.md). No implementation
exists yet — rules here are durable defaults grounded in those documents, not
inferred from code. Where a mechanism or vendor is unnamed, it is marked
`UNKNOWN` / `TO BE DECIDED` rather than invented.

This project is a **mobile-first web + API application** (web client → Node.js REST API
→ data stores, deployed via Terraform). Native applications are out of scope for v1.
API, browser, operational, and data-protection controls are all part of the security boundary.

---

## Required Security Inputs

Needed before these rules can be made concrete. Until supplied, treat as `UNKNOWN`.

- **Data store(s) and search engine** — `TO BE DECIDED`. Drives parameterized-query
  and injection-defense specifics (SEC-005).
- **Hosting / cloud provider and network topology** (behind Terraform) — `TO BE DECIDED`.
  Drives secrets management, network segmentation, WAF/rate-limit placement.
- **Secrets manager / KMS** for field-level address encryption (DATA-LOC-003) — `UNKNOWN`.
- **IAL2 identity-proofing vendor** (ORG-003) — `TO BE DECIDED`; must stay vendor-agnostic.
- **Push and email providers** (NOTIFY) — `TO BE DECIDED`.
- **Web session implementation** — `TO BE DECIDED`; if cookies are used they require
  CSRF protection and secure cookie attributes. Browser storage shall not contain bearer
  tokens or other long-lived credentials.
- **Vulnerability scanning / SCA tooling** and CI security gates — `UNKNOWN`.

---

## Provisional Security Rules

Binding until superseded by a dedicated security-architecture pass. Each maps to a
requirement where one exists.

### Authorization (AUTHZ-001..003, EVT-003)
- Enforce authorization **server-side on every protected request**; deny by default
  on missing or ambiguous state. Clients are untrusted and hold no authoritative state.
- No IDOR: object-ID manipulation never grants access. Hidden UI is not access control.
- Co-organizer permissions are evaluated **per event and per action**; attendees reach
  only their own RSVP data; each restricted-location read requires explicit authorization.
- Moderator/admin permissions are separate from organizer permissions; high-risk
  moderator actions require a documented reason.

### Authentication & Sessions (SEC-001/002, ARCHITECTURE auth model)
- HTTPS everywhere; credentials and session identifiers never appear in URLs.
- Passkeys (FIDO2/WebAuthn) for admins; password + optional MFA for others, with **MFA
  required** for moderators, admins, and Verified organizers. OIDC social login
  (Google, Facebook, Apple) — validate `iss`/`aud`/`nonce`, never trust client-sent identity.
- Sessions per OWASP ASVS; server-side invalidation on logout, password change, and
  suspension. Cookie sessions: `Secure`, `HttpOnly`, `SameSite`. Store no long-lived
  authentication secrets out of browser-accessible storage.
- Recovery, identifier changes, MFA reset/enrollment, and identity linking follow
  SEC-011..014: generic responses, short-lived single-use tokens, independent notification,
  current-password or equivalent proof, recent step-up, session invalidation, and no
  support override. OIDC identity keys are `(issuer, subject)`, never email alone.
- Rotate session identifiers after authentication and privilege change; enforce idle and
  absolute expiry; bind privilege decisions to current server state rather than token claims
  that can outlive revocation.

### Input, Output & Injection (SEC-004/005/006)
- Parameterized/prepared queries only; never build queries by string concatenation.
- Contextual output encoding for all user-controlled text. No raw HTML in descriptions;
  Markdown goes through an approved parser **and** sanitizer (allowlist).
- Validate all external URLs before display or redirect. **No server-side fetch** of
  organizer-supplied URLs unless explicitly required and SSRF-protected (allowlist,
  block internal/link-local ranges, no redirects to private space).
- Ticket links limited to the admin allowlist of established domains (FRAUD-004).

### Location Privacy & Data Protection (LOC, HOME, DATA-LOC, PRIV)
- Restricted/exact location data never crosses a boundary toward an unauthorized
  consumer: not via API responses, search, client state, map markers, EXIF, exports,
  notification previews, analytics, or error messages (LOC-004). Discovery/search paths
  must not even load exact private-location data (SEARCH-002).
- Location release is re-authorized **at release time**, not RSVP time; revoked RSVP,
  suspension, or holds block release (LOC-006).
- Exact private addresses: field-level encrypted at rest, access-logged per read,
  excluded from ordinary logs, deleted/de-identified ≤90 days post-event (DATA-LOC-003/004).
- Strip media metadata/EXIF (including geolocation) before publication; store media
  outside executable paths, access-controlled by event visibility (MEDIA-002, LOC-007).
- Data minimization; precise location never sold; private-event addresses never used for
  ads or profiling (PRIV-001/002).

### Platform Hardening (SEC-003/007/008/009/010)
- CSRF protection on all state-changing requests **when** cookie-based auth is used.
- No secrets in source code; use a secrets manager. Dependencies versioned and
  continuously assessed for known vulnerabilities (see Dependency Rules).
- Rate-limit registration, auth, password reset, organizer applications, event creation,
  RSVPs, reports, uploads, and notification triggers.
- Generic user-facing errors for security failures; details only in protected logs.
  Production logs exclude passwords, tokens, full private addresses, identity documents,
  and unnecessary PII.
- Automated tests covering authorization, input handling, and workflow security are
  required, not optional.
- Enforce per-route body, header, parameter, pagination, upload, decompression, execution,
  and concurrency bounds. Rate controls combine per-account, per-source, per-object, and
  global quotas and return bounded responses without revealing account existence.
- Allowlist production origins and hosts; reject credentialed wildcard CORS. Apply a
  restrictive CSP, anti-framing policy, MIME sniffing protection, referrer policy, and
  `Cache-Control: private, no-store` to authenticated or restricted responses. Service
  workers must never cache restricted or authenticated API data.

### Browser Client
- Prefer server-managed `Secure`, `HttpOnly`, appropriately scoped `SameSite` cookies;
  never put session credentials in URLs, localStorage, analytics, or error reports.
- State-changing requests require CSRF tokens plus Origin/Referer validation where
  cookie authentication is used. GET/HEAD/OPTIONS remain side-effect free.
- DOM insertion uses safe framework primitives and contextual encoding. Any HTML/Markdown
  sanitization is centralized, tested against mutation XSS, and paired with CSP; URL schemes
  and redirect destinations are allowlisted.
- Sensitive views do not prefetch, prerender, persist in browser history where avoidable,
  or expose values to third-party scripts. Third-party scripts are minimized and isolated.

### Integrations, Webhooks & Asynchronous Work
- Treat every provider assertion and callback as untrusted input. Verify signature over raw
  bytes, timestamp/freshness, nonce/event ID, expected issuer/audience/tenant, schema, and
  business context before recording it. Reject unsigned, stale, replayed, or ambiguous data.
- Outbound HTTP uses explicit destination allowlists, DNS/IP validation at connection time,
  redirect restrictions, short timeouts, response-size caps, and blocked loopback/private/
  link-local/metadata ranges. Credentials are scoped per provider and never forwarded.
- Jobs and events are versioned, minimal, authenticated where boundaries require it, and
  idempotent. Workers reauthorize restricted-location release and other sensitive effects
  using current state, not enqueue-time claims.

### Cryptography, Secrets, Backups & Deletion
- Use platform KMS/secrets management and envelope encryption with authenticated encryption
  and record/context binding. Key IDs/versions accompany ciphertext; rotation supports safe
  re-encryption. Key use, policy changes, disablement, and decrypts of restricted data audit.
- Secrets never enter source, build logs, Terraform state, client bundles, crash reports, or
  general environment dumps. Rotation and emergency revocation are tested.
- Backups are encrypted, access-separated, restore-tested, retention-bounded, and included
  in deletion/legal-hold procedures. A restore is quarantined until deletions, revocations,
  and current policy are reconciled.

### Integrity & Audit (REL-001, AUDIT-001..003)
- Material changes commit **before** notifications; notification failure never rolls back
  a valid change.
- Every sensitive action writes an append-only, tamper-protected, access-restricted audit
  entry: actor, action, target, timestamp, result, old/new values, reason where required,
  correlation ID.
- The mutation and audit/outbox record commit atomically. If durable audit capture is
  unavailable, sensitive mutations fail closed. Audit records use synchronized time,
  distinguish user/service identities, are integrity-monitored, and cannot be edited by
  application or moderator roles. Audit reads/exports/configuration changes are audited.

## STRIDE Threat Register

| ID | Category | Threat / affected assets | Required controls | Verification |
|---|---|---|---|---|
| S-01 | Spoofing | Credential stuffing or recovery takeover of attendee/organizer accounts | SEC-011/012, generic responses, throttling, session invalidation | Enumeration and recovery replay tests |
| S-02 | Spoofing | OIDC mix-up, email-based account collision, forged provider callback | SEC-013/016, issuer+subject key, state/nonce/PKCE, signed fresh callback | Negative callback/linking suite |
| S-03 | Spoofing | Staff/service impersonation or shared privileged identity | Unique identities, phishing-resistant MFA, managed short sessions, no impersonation | IAM and audit review |
| T-01 | Tampering | Mass assignment changes role, owner, status, visibility, or capacity | Command allowlists, server-derived fields, schema validation | Property-level API tests |
| T-02 | Tampering | Concurrent/stale/replayed mutation corrupts RSVP, revision, or notification state | SEC-015, atomic writes, versions, unique constraints, idempotency | Race/replay/fault tests |
| T-03 | Tampering | Audit, queue, policy, allowlist, or backup alteration | Transactional outbox, integrity protection, reviewed/versioned config, separated access | Reconciliation and restore tests |
| R-01 | Repudiation | Actor denies moderation, export, address access, role change, or message | AUDIT-001..003, purpose/reason, correlation and delivery evidence | Audit completeness tests |
| I-01 | Information Disclosure | Private address leaks through API/search/cache/HTML/map/telemetry/export | Data separation, response shaping, no-store, no prefetch, access-per-read audit | Canary-field and cache tests |
| I-02 | Information Disclosure | Reporter, attendee, identity proofing, or raw media data leaks | Minimization, field access policy, EXIF stripping, export controls, retention deletion | Role matrix and data-flow tests |
| I-03 | Information Disclosure | Logs/backups/errors/secrets reveal protected data | Redaction, encryption, separated access, secret scanning, restore quarantine | Log scanning and restore exercise |
| D-01 | Denial of Service | Auth/report/search/upload floods or parser/decompression bombs | SEC-018 layered quotas and hard resource bounds | Load, oversized-input, slow-client tests |
| D-02 | Denial of Service | Notification fan-out, queue poison, or provider outage exhausts workers | Bounded queues/retries, DLQ, backpressure, circuit breakers, idempotency | Dependency-failure drills |
| D-03 | Denial of Service | Lockout abuse blocks victim accounts | Progressive risk controls without permanent attacker-triggered lockout | Distributed-attempt tests |
| E-01 | Elevation of Privilege | BOLA/BFLA/IDOR or co-organizer scope escape | Per-request/object/action authorization and deny-by-default response shaping | Full role/object matrix tests |
| E-02 | Elevation of Privilege | Moderator/admin self-approval, stale grant, or stolen session | SEC-014 step-up, separation of duties, dual control, current-state reauth | Privileged workflow tests |
| E-03 | Elevation of Privilege | Callback or worker directly grants trust/release | Integration gateway; domain-policy reevaluation at moment of effect | Forged/stale-event tests |

Residual risk is not accepted implicitly. Before implementation, each `TO BE DECIDED`
technology choice shall be checked against this register; any control that cannot be met
requires a documented owner, compensating control, expiry date, and explicit approval.

---

## Prompt Placeholders To Resolve

The architecture identifies the stack, so these are **resolved**. Compressed, high-signal
rules for Claude Code follow each; the authoritative reference is linked, not copied.

- `{{CODE_QUALITY_PROMPT}}` → **RESOLVED.** Keep cyclomatic and cognitive complexity low;
  enforce separation of concerns; small pure functions; no god modules. Security logic
  (authz, crypto, validation) lives in dedicated, individually testable units.

- `{{API_SECURITY_PROMPT}}` → **RESOLVED** (Node.js REST).
  Refs: <https://owasp.org/www-project-api-security/>,
  <https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html>.
  Object- and function-level authorization on every endpoint (no BOLA/BFLA); explicit
  allowlist input validation and response shaping (no mass assignment, no over-exposure of
  fields); rate limiting and resource quotas; correct HTTP verbs/status; security headers;
  version and inventory all endpoints.

- `{{BACKEND_FRAMEWORK_PROMPT}}` → **RESOLVED** (Node.js).
  Ref: <https://nodejs.org/learn/getting-started/security-best-practices/>.
  Run with least privilege; validate/escape all inputs; avoid `child_process`/`eval` on
  user data; set secure HTTP headers; keep the runtime and deps patched; never leak stack
  traces; protect against prototype pollution and ReDoS.

- `{{FRONTEND_FRAMEWORK_PROMPT}}` → **PARTIALLY RESOLVED** (mobile-first web; framework
  `TO BE DECIDED`). Apply the Browser Client rules above independent of framework:
  server-managed sessions, CSRF protection, contextual encoding, CSP, restricted CORS,
  safe caching, dependency minimization, and no credentials in browser-readable storage.

- `{{AUTH_PROMPT}}` → **RESOLVED** (passkeys + password/MFA + OIDC).
  Ref: <https://fidoalliance.org/passkeys/>.
  Passkeys/WebAuthn for admins; phishing-resistant MFA where required; validate OIDC
  tokens fully; ASVS-conformant session lifecycle with server-side invalidation.

- `{{DEPLOYMENT_PROMPT}}` → **RESOLVED** (Terraform IaC).
  Ref: <https://www.wiz.io/academy/application-security/terraform-security-best-practices/>.
  No secrets in state or `.tf`; remote encrypted state with locking and least-privilege
  access; scan IaC in CI (tfsec/Checkov-class); least-privilege IAM; immutable, reviewed,
  version-pinned modules.

---

## Selected Prompt Imports (derived from architecture)

| Decision area | Choice (per ARCHITECTURE.md / REQUIREMENTS.md) | Prompt / standard |
|---|---|---|
| Architecture | 3-tier RN → REST API → data stores, server-authoritative | Code quality + API security rules above |
| Backend framework | Node.js | Node.js security best practices |
| Frontend framework | Mobile-first web; framework `TO BE DECIDED` | Browser Client rules above |
| Auth model | Passkeys (admin) + password/MFA + OIDC | FIDO passkeys + OWASP ASVS sessions |
| Deployment model | Terraform IaC | Terraform security best practices |
| Data store / search | `TO BE DECIDED` | Unresolved — parameterized-query rule applies generically |
| Hosting / cloud | `TO BE DECIDED` | Unresolved — secrets, network, WAF placement pending |
| Identity proofing (IAL2) | `TO BE DECIDED` (vendor-agnostic) | Unresolved |

---

## Dependency Rules

- Do not add a dependency when the standard library or a few lines of first-party code
  will do.
- Prefer **zero** new dependencies. If a library is required, justify it in the PR description.
- Only actively maintained libraries (a commit or release within the last 12 months).
- Only the latest stable major version. No deprecated, abandoned, or pre-release packages.
- Reject any library with known unpatched CVEs — check before adding and on every update.
- Audit transitive dependencies, not just direct ones. A small direct dep with a large or
  unvetted tree is a rejection.
- Pin exact versions with a committed lockfile. No floating ranges in production.
- Prefer narrow-scope libraries with minimal dependencies of their own and a clear
  security track record.
