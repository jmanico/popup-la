# SECURITY.md — PopUp-LA

Secure-coding rules and security guardrails for PopUp-LA. Sources of truth:
[REQUIREMENTS.md](REQUIREMENTS.md) (§19–22 AUTHZ/SEC/AUDIT/PRIV are binding),
[ARCHITECTURE.md](ARCHITECTURE.md), and [DESIGN.md](DESIGN.md). No implementation
exists yet — rules here are durable defaults grounded in those documents, not
inferred from code. Where a mechanism or vendor is unnamed, it is marked
`UNKNOWN` / `TO BE DECIDED` rather than invented.

This project is a **mobile + API application** (React Native iOS/Android clients →
Node.js REST API → data stores, deployed via Terraform). Priorities, in order:
**API security**, then **mobile security**, then data protection and privacy.

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
- **CSRF model decision** — cookie-based sessions vs. bearer tokens for the mobile
  clients (changes whether SEC-003 CSRF tokens apply). Currently `UNKNOWN`.
- **Native vs. web client scope** — REQUIREMENTS.md §1 (web) vs. ARCHITECTURE.md
  (native RN) conflict, `TO BE DECIDED`. Mobile rules below assume native RN.
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
  secrets in RN `AsyncStorage`; use Keychain/Keystore-backed secure storage.

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

### Mobile (React Native)
- Treat the device as hostile: no trust in client-side checks, no sensitive business
  logic or secrets shipped in the bundle.
- Secure local storage via Keychain (iOS) / Keystore (Android); never plaintext.
- Certificate pinning for the API where feasible; enforce TLS, reject cleartext traffic.
- Guard deep links and inter-app URLs; validate all inbound parameters server-side.

### Integrity & Audit (REL-001, AUDIT-001/002)
- Material changes commit **before** notifications; notification failure never rolls back
  a valid change.
- Every sensitive action writes an append-only, tamper-protected, access-restricted audit
  entry: actor, action, target, timestamp, result, old/new values, reason where required,
  correlation ID.

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

- `{{FRONTEND_FRAMEWORK_PROMPT}}` → **RESOLVED** (React Native).
  Ref: <https://reactnative.dev/docs/security>.
  Secure storage (Keychain/Keystore); no secrets in the bundle; TLS + optional cert
  pinning; safe deep-link handling; least-privilege native permissions; sanitize any
  WebView/HTML content.

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
| Frontend framework | React Native (native iOS/Android)¹ | React Native security |
| Auth model | Passkeys (admin) + password/MFA + OIDC | FIDO passkeys + OWASP ASVS sessions |
| Deployment model | Terraform IaC | Terraform security best practices |
| Data store / search | `TO BE DECIDED` | Unresolved — parameterized-query rule applies generically |
| Hosting / cloud | `TO BE DECIDED` | Unresolved — secrets, network, WAF placement pending |
| Identity proofing (IAL2) | `TO BE DECIDED` (vendor-agnostic) | Unresolved |

¹ Native-vs-web client scope is `TO BE DECIDED` (REQUIREMENTS.md §1 conflict); rules assume native RN.

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
