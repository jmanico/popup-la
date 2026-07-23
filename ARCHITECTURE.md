# ARCHITECTURE.md — PopUp-LA

Provisional, bootstrapped architecture. Sources of truth: [REQUIREMENTS.md](REQUIREMENTS.md)
(what the system must do) and [DESIGN.md](DESIGN.md) (visual/frontend design language the
client must support). No implementation exists yet; nothing here is inferred from code.

> ⚠️ Conflict to resolve: the human-provided runtime input below (native mobile apps,
> React Native) contradicts REQUIREMENTS.md §1, which lists "native apps" as **out of scope
> for v1** and describes a "mobile-first web platform." This document follows the explicit
> human input (native apps + REST API) and flags the conflict as `TO BE DECIDED`. The
> location-privacy, authorization, and honesty requirements are client-agnostic and apply
> regardless of which client wins.

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
Runtime environment:        Mobile app (Android and iOS), plus an API tier.
Server framework:           Node.js
Client framework:           React Native
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
Security expectations:      TO BE DECIDED — dedicated security architecture pass is next.
                            REQUIREMENTS.md §19–22 (AUTHZ, SEC, AUDIT, PRIV) are binding
                            constraints that pass must satisfy.
```

## Initial Architecture (Provisional)

A three-tier system: React Native clients → REST API (Node.js) → data stores. All
authorization, capacity, and location-release decisions are server-side; clients are
untrusted (EVT-003, AUTHZ-001).

**Clients (React Native, iOS + Android).** Render discovery, event detail, RSVP,
organizer, and moderation flows. Must express the DESIGN.md language (navy/sunset palette,
signature gradient, pin-as-hero, mobile-first touch layout) and meet WCAG 2.2 AA
(A11Y-001): no color-only meaning, accessible names, alt text. Clients hold no
authoritative state and never receive restricted location data they aren't authorized for
(LOC-004, SEARCH-002).

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

**Cross-cutting constraints (assumptions where a mechanism is unnamed):**
- Server-authoritative authorization, no IDOR, per-event/per-action co-organizer checks
  (AUTHZ-002/003).
- Location-release authorization is evaluated at release time, not RSVP time (LOC-006).
- Capacity check + RSVP confirmation in one atomic/concurrency-safe transaction (CAP-002).
- Material changes commit before notifications; notification failure never rolls back a
  valid change (REL-001).
- Audit writes accompany every sensitive action (AUDIT-001).

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
- Web vs. native client scope conflict — `TO BE DECIDED` (see top-of-file note).
- Full security architecture — `TO BE DECIDED` (next pass).

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
| Audit | AUDIT-001/002 |
| Cross-cutting authorization | AUTHZ-001..003, EVT-003 |
| Client UI/UX + design language | DESIGN.md, A11Y-001/002 |
| Data protection / privacy | PRIV-001/002, DATA-LOC-003/004, SEC-009 |
| Security architecture | SEC-001..010 — **TO BE DECIDED** (dedicated pass next) |
| Data store / hosting / vendor selection | **TO BE DECIDED** (no vendor in REQUIREMENTS.md) |
| Native vs. web client scope | REQUIREMENTS.md §1 vs. human input — **TO BE DECIDED** |

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
