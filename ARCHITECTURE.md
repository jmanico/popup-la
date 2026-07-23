# ARCHITECTURE.md — PopUp-LA

Provisional, bootstrapped architecture. Sources of truth: [REQUIREMENTS.md](REQUIREMENTS.md)
(what the system must do) and [DESIGN.md](DESIGN.md) (visual/frontend design language the
client must support). No implementation exists yet; nothing here is inferred from code.

The v1 client is the mobile-first web application defined by REQUIREMENTS.md §1. Earlier
React Native input is retained only as a possible post-v1 option. Security boundaries are
client-agnostic, but browser-specific session, CSRF, CORS, CSP, and cache controls are
mandatory for v1.

## Required Architecture Inputs

```
Requirements source:        REQUIREMENTS.md
System purpose:             Mobile platform for discovering and publishing temporary events in
                            Los Angeles, with community safety, accurate listings, privacy-
                            preserving location handling, vetted organizer access, transparent
                            moderation, and secure-by-default implementation as first priorities.
Primary use cases:          Browse/search public event listings; RSVP and manage RSVPs; apply
                            for and hold organizer status (identity-proofed); create/publish/
                            edit/cancel events; privacy-preserving location + private-residence
                            handling; co-organizer collaboration; duplicate/fraud detection;
                            moderation, reporting, and appeals; audited administration.
Target users / actors:      Visitor, Attendee, Organizer Applicant, Approved Organizer,
                            Co-Organizer, Moderator, Administrator (REQUIREMENTS.md §2).
Runtime environment:        Mobile-first web client plus an API tier.
Server framework:           Node.js
Client framework:           TO BE DECIDED (web framework); React Native is out of scope for v1.
API style / integration:    REST
Auth / session model:       Passkeys for admins; password + optional MFA for everyone else
                            (MFA REQUIRED for moderators, admins, and Verified organizers per
                            SEC-002); OIDC social login (Google, Facebook, Apple). Sessions
                            per OWASP ASVS; server-side invalidation on logout/password-change/
                            suspension (SEC-002).
Data model expectations:    Derived from REQUIREMENTS.md — core entities below. Server-
                            authoritative for ownership/role/status/capacity/moderation state
                            (EVT-003). Field-level encryption for exact private addresses
                            (DATA-LOC-003).
Deployment model:           Terraform (infrastructure as code).
Scale expectation:          Los Angeles only (single-region, single-metro).
Security expectations:      STRIDE threat model and binding controls are defined below and
                            in SECURITY.md. Concrete products remain TO BE DECIDED.
```

## Initial Architecture (Provisional)

A three-tier system: web clients → REST API (Node.js) → data stores. All
authorization, capacity, and location-release decisions are server-side; clients are
untrusted (EVT-003, AUTHZ-001).

**Client (mobile-first web).** Renders discovery, event detail, RSVP,
organizer, and moderation flows. Must express the DESIGN.md language (navy/sunset palette,
signature gradient, pin-as-hero, mobile-first touch layout) and meet WCAG 2.2 AA
(A11Y-001): no color-only meaning, accessible names, alt text. Clients hold no
authoritative state and never receive restricted location data they aren't authorized for
(LOC-004, SEARCH-002). It receives only response-shaped data appropriate to the current
principal. Restricted responses are private/no-store and never embedded in initial HTML,
service-worker caches, prefetches, telemetry, or shared caches.

**API tier (Node.js, REST).** Stateless request handlers enforcing deny-by-default
authorization on every protected request (AUTHZ-001). Responsibilities: authN/session,
event lifecycle, atomic capacity/RSVP transactions (CAP-002), location-release evaluation
at release time (LOC-006), material-edit/revision handling (EDIT), moderation actions
(MOD), duplicate/fraud scoring (DUPE/FRAUD), and emitting audit records (AUDIT).

**Suggested service boundaries** (logical; deployment topology TO BE DECIDED):
- **Identity & Access** — registration, login (passkey/password/MFA/OIDC), roles, session
  lifecycle, organizer status state machine (ORG-004).
- **Organizer Verification** — vendor-agnostic IAL2 identity-proofing integration
  (ORG-003); stores verification result, deletes raw ID/biometric artifacts within 30 days
  (ORG-007); isolates PII.
- **Events & Discovery** — event CRUD, status machine (EVT-002), search/filter (SEARCH),
  ranking with penalties (EVT-005), duplicate evaluation (DUPE).
- **Location Privacy** — visibility-mode resolution, coordinate offsetting (LOC-005),
  release authorization re-check (LOC-006), private-address encryption + access logging
  (DATA-LOC-003).
- **Capacity & RSVP** — atomic capacity enforcement, waitlist, one-active-RSVP invariant
  (CAP/RSVP).
- **Moderation, Reports & Appeals** — report intake, risk scoring, moderator field-change
  boundaries (MOD), appeals SLAs (APPEAL-001).
- **Media** — upload validation (signature/type/size), metadata/EXIF stripping (MEDIA,
  LOC-007), access-controlled storage outside executable paths.
- **Notifications** — email (required) + optional push; delivery-attempt recording;
  previews that omit sensitive content (NOTIFY).
- **Audit** — append-only, tamper-protected, access-restricted event log (AUDIT).
- **Security Policy Enforcement** — centralized authentication/session validation,
  step-up checks, authorization policy, request validation, rate/quota enforcement, and
  security event emission. Application handlers cannot bypass this boundary.
- **Integration Gateway** — outbound provider calls and inbound webhooks/callbacks for
  OIDC, identity proofing, email, and push. It verifies source/signature/freshness/context,
  provides idempotency, constrains egress, and translates vendor state into untrusted input
  for domain policy evaluation.
- **Key and Secrets Boundary** — KMS/secrets-manager-backed envelope encryption, key
  versioning/rotation, separation of duties, and audited decrypt operations. Exact private
  locations and verification data use separate keys and access policies.

**Cross-cutting constraints (assumptions where a mechanism is unnamed):**
- Server-authoritative authorization, no IDOR, per-event/per-action co-organizer checks
  (AUTHZ-002/003).
- Location-release authorization is evaluated at release time, not RSVP time (LOC-006).
- Capacity check + RSVP confirmation in one atomic/concurrency-safe transaction (CAP-002).
- Material changes commit before notifications; notification failure never rolls back a
  valid change (REL-001).
- Audit writes accompany every sensitive action and are durably coupled to the mutation
  (AUDIT-001..003).
- Sensitive state changes and audit/outbox records share a transaction boundary; consumers
  are at-least-once and idempotent. Version checks prevent stale writes and replay.
- Authorization uses current server state at the moment of use. Cached grants, roles,
  RSVP state, or moderation decisions never authorize sensitive actions after revocation.
- Dependency failure is bounded by timeouts, circuit breakers, queues with finite capacity,
  and backpressure. Security dependencies fail closed; public read paths may degrade to
  explicitly safe, non-sensitive cached data.

## Trust Boundaries and Data Flows

1. **Internet/browser → edge/API:** untrusted HTTP, cookies, headers, files, URLs, and
   identifiers cross origin and authentication boundaries. TLS, trusted-host/CORS/CSP,
   CSRF, body limits, schema validation, authentication, authorization, and quotas apply.
2. **API → domain/data stores:** only server-derived identity and allowlisted fields cross.
   Queries are parameterized; row/object authorization and response shaping occur before
   access. Restricted location data is isolated from discovery/search storage.
3. **API → asynchronous queue/workers:** messages contain minimal references, integrity
   metadata, schema versions, correlation/idempotency keys, and authorization-relevant
   versions. Workers re-check current policy before sensitive effects.
4. **Platform → third parties:** identity, OIDC, mail, push, map, and ticket providers are
   independently untrusted. Egress is allowlisted; disclosures are minimized; callbacks
   are authenticated and replay-resistant; provider assertions never bypass domain rules.
5. **Operations/moderation → privileged interfaces:** separate roles, phishing-resistant
   MFA, recent step-up, managed sessions, no shared accounts, purpose binding, and complete
   audit apply. Production data access is exceptional and time-bounded.
6. **Data stores → backups/logs/analytics/search:** classification and minimization persist
   across derived stores. Restricted data is excluded by construction, encrypted where
   retained, and subject to deletion/legal-hold propagation and restore reconciliation.

## STRIDE Architecture Review

| Category | Principal threats | Required architectural mitigations |
|---|---|---|
| Spoofing | Credential stuffing, recovery takeover, OIDC mix-up/linking, forged callbacks, shared staff accounts | Phishing-resistant privileged MFA, secure recovery, issuer+subject identity keys, signed fresh callbacks, unique service/user identities, step-up |
| Tampering | Mass assignment, stale/concurrent edits, RSVP races, queue replay, policy/config or audit alteration | Allowlisted commands, atomic/versioned writes, idempotency, transactional outbox, signed/versioned configuration, tamper-evident audit |
| Repudiation | Denied moderation/export/location access, ambiguous service actions, notification disputes | Durable correlated audit, actor/service identity, reason/purpose fields, delivery evidence, synchronized time, audit-of-audit access |
| Information disclosure | Private addresses in search/cache/HTML/telemetry/backups; reporter or verification data leakage | Physical/logical data separation, response shaping, private/no-store caching, field encryption, metadata stripping, minimization and deletion propagation |
| Denial of service | Auth/report/upload/search flooding, decompression/parser abuse, notification fan-out, dependency exhaustion | Layered quotas, strict size/time/concurrency bounds, bounded queues, backpressure, circuit breakers, safe degradation |
| Elevation of privilege | IDOR/BFLA, forged roles, co-organizer scope escape, moderator self-approval, stale authorization | Central deny-by-default policy, per-object/action checks, current-state reauthorization, separation of duties, step-up and dual control |

**Core data entities (derived, not schema-final):** User/Account, OrganizerProfile +
VerificationRecord, Event + EventRevision, Location (public display + restricted
exact/encrypted), Venue (reusable public records), RSVP, Waitlist entry, Co-Organizer grant,
Report, ModerationAction, AppealCase, MediaAsset, Notification/DeliveryAttempt, AuditEntry,
AdminConfig (categories/neighborhoods/policy/ticket-allowlist).

**Explicit unknowns (not filled with guesses):**
- Data store(s) and search engine — `TO BE DECIDED` (REQUIREMENTS.md names no vendor).
- Hosting/cloud provider and network topology behind Terraform — `TO BE DECIDED`.
- Push provider, email provider, IAL2 verification vendor — `TO BE DECIDED` (must stay
  vendor-agnostic per ORG-003).
- Deployment topology (modular monolith vs. services) — `TO BE DECIDED`; boundaries above
  are logical only.
- Web framework and browser rendering/deployment approach — `TO BE DECIDED`; v1 remains web.
- Concrete security products/vendors — `TO BE DECIDED`; the controls and trust boundaries
  in this document are binding regardless of product choice.

## Requirement Traceability

| Architecture component / boundary | Requirement groups |
|---|---|
| Identity & Access (auth, sessions, roles, org status) | §2 Roles, ORG-001/004, RSVP-001, SEC-001/002/003, AUTHZ-003 |
| Organizer Verification service | ORG-002/003/006/007, PRIV-002 |
| Events & Discovery (CRUD, status, search, ranking, dupes) | EVT-001..005, SEARCH-001/002, DUPE-001..003 |
| Location Privacy service | LOC-001..007, HOME-001..004, DATA-LOC-001..004, SEARCH-002 |
| Capacity & RSVP service | CAP-001..005, RSVP-001..003, PERF-001 |
| Editing & revisions | EDIT-001..004, EVT-002 |
| Cancellation | CANCEL-001..004 |
| Co-organizer collaboration | COLLAB-001..003, AUTHZ-002 |
| Moderation, Reports & Appeals | MOD-001..005, FRAUD-001..006, REPORT-001, APPEAL-001 |
| Media service | MEDIA-001/002, LOC-007 |
| Notifications | NOTIFY-001..003, REL-001 |
| Audit | AUDIT-001..003 |
| Cross-cutting authorization | AUTHZ-001..003, EVT-003 |
| Client UI/UX + design language | DESIGN.md, A11Y-001/002 |
| Data protection / privacy | PRIV-001/002, DATA-LOC-003/004, SEC-009 |
| Security architecture | SEC-001..020, AUDIT-001..003, PRIV-001..004; STRIDE review below |
| Data store / hosting / vendor selection | **TO BE DECIDED** (no vendor in REQUIREMENTS.md) |
| Web client | REQUIREMENTS.md §1; framework remains **TO BE DECIDED** |

## Dependency Rules

1. **Server is authoritative; clients are untrusted.** Ownership, role, status, capacity,
   and moderation state are decided server-side. Client-supplied values never override
   server authorization (EVT-003, AUTHZ-001).
2. **Every protected request enforces authorization independently.** Deny by default on
   missing/ambiguous state. Hidden UI is not an access control; object-ID manipulation
   never grants access (AUTHZ-001/002/003).
3. **Restricted location data never crosses a boundary toward an unauthorized consumer.**
   Not via API responses, search, client state, metadata, map markers, EXIF, exports,
   notification previews, analytics, or errors (LOC-004). Discovery/search paths must not
   even load exact private-location data (SEARCH-002).
4. **Location release depends on a fresh authorization check.** Re-evaluated at release
   time; RSVP revocation, suspension, or holds block release (LOC-006).
5. **Capacity mutations depend on an atomic transaction.** Capacity check and RSVP
   confirmation are one concurrency-safe operation; confirmed attendance never exceeds
   capacity (CAP-002).
6. **State changes depend on (precede) notifications, never the reverse.** Material change
   commits first; notification failure never rolls back a valid change (REL-001).
7. **Every sensitive action depends on an audit write.** Approvals, publication, material
   edits, cancellations, visibility/location access, capacity/permission changes,
   moderation, fraud/dupe decisions, removals, exports (AUDIT-001).
8. **PII and verification data are isolated and minimized.** Raw ID/biometric artifacts
   are short-lived (ORG-007); exact private addresses are encrypted and access-logged
   (DATA-LOC-003); verification data is never public by default (PRIV-002).
9. **External-content boundaries are guarded.** Ticket links limited to an admin allowlist
   (FRAUD-004); organizer-supplied URLs validated, no server-side fetch unless explicitly
   required and SSRF-protected (SEC-006); user text contextually encoded, Markdown
   sanitized (SEC-004).
10. **Moderation is bounded.** Moderators change only system-controlled or evidence-backed
    structured fields and never silently rewrite organizer-authored content (MOD-003/005).
