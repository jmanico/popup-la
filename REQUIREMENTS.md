# REQUIREMENTS.md — PopUp-LA

## 1. Overview

PopUp-LA is a mobile-first web platform for discovering and publishing temporary events in Los Angeles. Priorities: community safety, accurate listings, privacy-preserving location handling, vetted organizer access, transparent moderation, secure-by-default implementation.

**Out of scope (v1):** native apps, payment processing, ticket resale, public attendee lists, attendee-to-attendee messaging, real-time location tracking, background checks, emergency dispatch, safety guarantees. The v1 client is a mobile-first web application; native-client architecture is a possible future extension and is not an implementation requirement.

## 2. Roles

| Role | Summary |
|---|---|
| Visitor | Browse, search, view public listings and organizer profiles, register |
| Attendee | RSVP, cancel own RSVP, save events, follow organizers, report, manage profile/notifications |
| Organizer Applicant | Apply, submit verification, respond to moderators. May NOT publish |
| Approved Organizer | Create/publish/edit/cancel own events, invite co-organizers, view RSVP counts, send event updates |
| Co-Organizer | Only permissions explicitly granted per event |
| Moderator | Review applications/reports/duplicates; restrict, hide, suspend, restore; change moderation-controlled fields only |
| Administrator | Roles, categories, neighborhoods, policy configuration. No audit bypass |

## 3. Organizer Eligibility (ORG)

- **ORG-001** Any registered user may apply for organizer status; only approved organizers may publish.
- **ORG-002** Application requires: verified email, verified phone, display name, description, contact info, terms acceptance, accuracy attestation.
- **ORG-003** Identity proofing shall meet NIST SP 800-63A IAL2 equivalence (government ID + liveness match) via a vendor-agnostic integration. Risk-based additions: business registration, venue confirmation, org-domain email, manual review.
- **ORG-004** Statuses: Applicant, Limited, Approved, Verified, Restricted, Suspended, Revoked. All transitions audit-logged.
- **ORG-005** New organizers start as Limited. Limited-trust organizers require manual review for: their first 3 published events, every private-residence event, any paid event or event with external payment links, capacity > 50, and material edits within 24 hours of start time.
- **ORG-006** Verified status requires: IAL2 proofing, 3 completed events with no substantiated reports, and venue/business affiliation confirmation.
- **ORG-007** Verification results (pass/fail, method, date, vendor reference) retained for account lifetime + 1 year. Raw identity documents and biometric artifacts deleted within 30 days of decision.
- **ORG-008** Approval shall never be presented as a guarantee of event safety, legality, or quality.

## 4. Events (EVT)

- **EVT-001** Required fields to publish: title, description, category, organizer, start/end (start < end), time zone, location type, visibility mode, capacity mode (+ max where applicable), RSVP policy, age guidance, accessibility info.
- **EVT-002** Statuses: Draft, Pending review, Published, Paused, Sold out, Canceled, Completed, Removed. Draft/pending never appear in public discovery.
- **EVT-003** Ownership, role, status, capacity, and moderation state are server-authoritative. Client-supplied values for these fields shall never override server-side authorization.
- **EVT-004** Titles shall not use impersonation, keyword stuffing, false urgency, or misleading formatting. Descriptions shall not conceal material terms (mandatory fees, age limits, entry requirements, accessibility limits, location uncertainty).
- **EVT-005** Discovery ranking may be lowered for substantiated complaints, frequent cancellations, unresolved duplicates, misleading or incomplete content. Ranking penalties never silently alter event content. Payment shall never override safety restrictions; sponsored placement, if added, is clearly labeled.

## 5. Location Privacy (LOC)

- **LOC-001** Location types: public venue, outdoor public space, temporary commercial venue, private residence, mobile, online, announced later.
- **LOC-002** Visibility modes: exact public, approximate public, neighborhood only, exact after approved RSVP, timed release, online link after RSVP, moderator-restricted.
- **LOC-003** Private-residence exact addresses are never public by default. Public page shows at most: neighborhood, approximate area, arrival guidance, organizer identity, release policy.
- **LOC-004** Restricted addresses shall not leak via: public APIs, search results, HTML source, client state, structured metadata, map markers, image EXIF, calendar exports, notification previews, analytics, or error messages.
- **LOC-005** Approximate map pins shall use genuinely offset coordinates, not exact coordinates with visual masking.
- **LOC-006** Address-release authorization shall be re-evaluated at release time, not only at RSVP time (RSVP revocation, suspension, or moderation holds block release).
- **LOC-007** All uploaded images are stripped of embedded metadata (including geolocation) before publication.

## 6. Private Residences (HOME)

- **HOME-001** Private residences are permitted only with: an Approved+ organizer, verified contact info, restricted address visibility, a defined attendee limit, mandatory organizer approval of each attendee before address release, a reporting mechanism, and acceptance of private-event safety terms.
- **HOME-002** Capacity caps by trust tier: Limited 10, Approved 25, Verified 50. Exceeding the tier cap requires moderator approval with documented rationale.
- **HOME-003** The platform may require proof of authorization to use the residence; moderators may prohibit an event for unclear authorization, unsafe capacity, suspected fraud, repeated substantiated complaints, or refused verification.
- **HOME-004** Listings never reveal the resident's name without explicit authorization. A private-location notice is displayed before RSVP. The platform never claims to inspect or certify residences.

## 7. Editing and Revisions (EDIT)

- **EDIT-001** Minor edits (typos, clarifications, images) require no notification. Material changes (date, time, venue/location, neighborhood, price or mandatory fee, age restriction, capacity reduction, cancellation policy, organizer identity, accessibility conditions, event purpose) create a new revision preserving prior values.
- **EDIT-002** Material changes notify all affected attendees, stating: what changed, previous value, new value, when, whether reconfirmation is required, and how to cancel.
- **EDIT-003** Reconfirmation is required for: date changes, neighborhood changes, material venue changes, free-to-paid, new exclusionary age restriction, or substantial change of event nature. Non-reconfirming attendees may move to pending/canceled after a configured period.
- **EDIT-004** An event shall not be edited into an unrelated event while retaining its RSVP list. Moderators may freeze editing during investigations.

## 8. Cancellation (CANCEL)

- **CANCEL-001** Cancellation requires a categorized reason; it stops new RSVPs, notifies all attendees, preserves the record and audit history, removes the event from upcoming results, and marks it Canceled everywhere it appears.
- **CANCEL-002** Canceled events are never silently deleted and cannot be restored to Published after attendee notification; resumption requires a new event or a duplicated draft.
- **CANCEL-003** Refunds are never stated as issued unless processed and verified by the platform.
- **CANCEL-004** Frequent cancellations may trigger ranking reduction, review, publication limits, or suspension.

## 9. Co-Organizers (COLLAB)

- **COLLAB-001** Every event has exactly one primary organizer; the last valid primary cannot be removed without transfer or cancellation.
- **COLLAB-002** Co-organizer invitations require explicit acceptance. Permissions are per-event, least-privilege, from: edit content, edit location, manage images, view RSVP totals, view approved attendee info, send notifications, manage capacity, cancel, invite/remove co-organizers.
- **COLLAB-003** Co-organizer access never extends to the primary's other events. Grants, changes, revocations, and significant actions are audit-logged. Primary organizers and moderators may revoke access.

## 10. Duplicates (DUPE)

- **DUPE-001** New/edited events are evaluated for duplicates (title, organizer, venue, coordinates, time overlap, description, ticket URL, images, contact info). Title similarity alone never auto-rejects.
- **DUPE-002** Likely duplicates: warn the organizer, show matches, allow explanation and draft-save; high-confidence cases route to moderation.
- **DUPE-003** Moderators may merge, select canonical, remove, link sessions, or request clarification. Merges preserve audit history, attribution, RSVPs, and notification obligations. Attendee data never crosses listings without authorization.

## 11. Fraud (FRAUD)

- **FRAUD-001** Report categories: fake event, impersonation, misleading location, hidden fees, payment fraud, dangerous activity, prohibited content, duplicate, unauthorized venue use, other.
- **FRAUD-002** Events receive a risk score from documented indicators (new account, unverified contacts, copied content/images, suspicious payment links, conflicting venue info, complaint volume, cancellation history, sudden high-capacity private events, moderation evasion).
- **FRAUD-003** High-risk events may be held, hidden from search/recommendations, blocked from new RSVPs, marked under review, or suspended. No public "fraudulent" label before a moderator determination.
- **FRAUD-004** External ticket links are restricted to an administrator-managed allowlist of established ticketing domains; non-allowlisted domains require moderator approval.
- **FRAUD-005** Moderators record evidence and rationale; evidence is retained per policy. Confirmed fraudulent organizers may be suspended, revoked, blocked from re-registration, and reported to providers or authorities where legally appropriate.
- **FRAUD-006** The platform shall not expose details that assist abuse, identity evasion, or retaliation against reporters.

## 12. Capacity and RSVP (CAP, RSVP)

- **CAP-001** Capacity modes: unlimited, fixed, approval required, waitlist, external-only. Fixed capacity requires a positive maximum.
- **CAP-002** Confirmed attendance is computed server-side from active RSVP records; capacity check and RSVP confirmation occur in one atomic transaction or equivalent concurrency-safe operation. Confirmed attendance never exceeds capacity.
- **CAP-003** One active RSVP per attendee per event. Cancellation releases capacity. Waitlist offers expire after a configured period.
- **CAP-004** Capacity reductions never auto-remove confirmed attendees; reductions below confirmed count require a reason, an explicit attendee-management plan, and moderator notification for substantial over-capacity.
- **CAP-005** Organizer-stated capacity is never presented as a governmental occupancy certification.
- **RSVP-001** RSVP requires an authenticated account with verified email; account minimum age 16.
- **RSVP-002** Organizers never create RSVPs on behalf of attendees. Attendee removal requires a documented reason, notifies the attendee, releases capacity, and is audit-logged.
- **RSVP-003** Attendee identities never appear publicly. Organizers receive only information necessary to operate the event. No sensitive attendee data in public exports or URLs.

## 13. Location Data Retention (DATA-LOC)

- **DATA-LOC-001** Retain only: public display location, neighborhood, city, postal code, venue ID, exact address/coordinates when required, visibility mode, release rules, revision history, verification state. Public venue addresses may persist as reusable venue records.
- **DATA-LOC-002** No continuous attendee location history; no attendee movement tracking. Search coordinates processed at minimum precision and retained only for completion, abuse prevention, and short-lived diagnostics. Analytics prefer coarse location.
- **DATA-LOC-003** Exact private addresses: field-level encrypted at rest, access logged per read, excluded from ordinary application logs, revision history restricted to authorized staff.
- **DATA-LOC-004** Absent an active report, dispute, investigation, or legal hold, private-event exact addresses are deleted or irreversibly de-identified no later than 90 days post-event, and no earlier than the end of the dispute period.

## 14. Moderation Boundaries (MOD)

- **MOD-001** Moderators may change system-controlled fields: moderation/publication/discovery status, trust status, duplicate/fraud/safety review states, age and content-warning classification, address-visibility restriction, RSVP and organizer suspension, removal reason, notes, appeal status.
- **MOD-002** Moderators may correct structured fields with clear evidence: category, neighborhood, venue classification, location type, accessibility classification, duplicate relationship, public/private designation.
- **MOD-003** Moderators shall not rewrite organizer-authored content (title, description, bio, claims, schedule, price, cancellation text) except to redact prohibited or imminently harmful content: private personal information, exposed private addresses, credible threats, harassment, illegal content, malicious links, exploitative imagery.
- **MOD-004** For inaccurate but non-dangerous content: restrict/flag if needed, request correction identifying the specific field, set a deadline, record the request. Emergency redactions preserve originals in restricted audit storage where legally permissible.
- **MOD-005** Every moderator change records actor, timestamp, field, old/new values, reason, supporting evidence, notification status, and appeal eligibility. Moderators never modify RSVP counts directly (fraudulent RSVPs invalidated only via documented actions) and never impersonate organizers.

## 15. Notifications (NOTIFY)

- **NOTIFY-001** Channels at launch: verified email (required); web/mobile push (optional). SMS deferred.
- **NOTIFY-002** Attendees are notified of: cancellation, material date/time/venue/neighborhood changes, address release, capacity status changes, removal, suspension, reconfirmation requirements. Delivery attempts for material notifications are recorded.
- **NOTIFY-003** Notifications never expose private addresses before the recipient is authorized; previews omit sensitive content. Operational channels are not usable for unrestricted promotion.

## 16. Reporting and Appeals (REPORT, APPEAL)

- **REPORT-001** Reports contain category, optional description/evidence, reporter ID, target, timestamp, status (Submitted, Under review, More info requested, Action taken, No violation, Duplicate, Closed, Appealed). Reporter identity is never disclosed to the reported party unless legally required or authorized. Rate limits prevent report flooding.
- **APPEAL-001** Organizers may appeal eligible decisions. Appeals acknowledged within 2 business days and decided within 7; suspension appeals decided within 3. Reviewed by a different moderator when practical; outcomes recorded and communicated.

## 17. Search (SEARCH)

- **SEARCH-001** Filters: keyword, date, category, neighborhood, approximate location, distance, organizer, accessibility, age suitability, availability.
- **SEARCH-002** Only published, discovery-eligible events appear in default results. Private-residence results never reveal exact addresses. Public search rendering never loads exact private-location data.

## 18. Media (MEDIA)

- **MEDIA-001** Enforce file-count, per-file size, and total limits; allowlisted media types; file-signature validation; safe server-generated filenames; rejection of unneeded active-content formats.
- **MEDIA-002** Metadata (including geolocation) stripped before publication. Media stored outside executable paths, access-controlled by event visibility, and removable by moderation.

## 19. Authorization (AUTHZ)

- **AUTHZ-001** Every protected request enforces server-side authorization; deny by default on missing or ambiguous state.
- **AUTHZ-002** Organizers access only owned or explicitly authorized events; co-organizer permissions evaluated per event and per action; attendees access only their own RSVP data; each access to a restricted location requires explicit authorization.
- **AUTHZ-003** Object-ID manipulation never grants access (no IDOR). Hidden UI is not an authorization control. Moderator/admin permissions are separate from organizer permissions. High-risk moderator actions require a documented reason.

## 20. Security (SEC)

- **SEC-001** HTTPS everywhere; credentials and session identifiers never in URLs.
- **SEC-002** Session management per OWASP ASVS: secure, HttpOnly, SameSite cookies; server-side invalidation on logout, password change, and suspension. MFA available to all users and required for moderators, administrators, and Verified organizers.
- **SEC-003** CSRF protection on all state-changing requests using cookie-based auth.
- **SEC-004** Contextual output encoding for all user-controlled text. No raw HTML in descriptions; any Markdown goes through an approved parser and sanitizer.
- **SEC-005** Parameterized database queries only.
- **SEC-006** External URLs validated before display or redirect; server-side fetching of organizer-supplied URLs prohibited unless explicitly required and SSRF-protected.
- **SEC-007** No secrets in source code. Dependencies versioned and continuously assessed for known vulnerabilities.
- **SEC-008** Rate limits on registration, authentication, password reset, organizer applications, event creation, RSVPs, reports, uploads, notification triggers.
- **SEC-009** Generic user-facing error messages for security failures; details in protected logs. Production logs exclude passwords, tokens, full private addresses, identity documents, and unnecessary PII.
- **SEC-010** Automated test suite covering authorization, input handling, and workflow security.
- **SEC-011** Registration, login, account recovery, email/phone change, OIDC linking, and MFA enrollment/reset shall resist account enumeration and takeover. Recovery shall use single-use, short-lived tokens, invalidate affected sessions after success, notify the account through an independently established channel, and require step-up authentication for security-sensitive changes. Support personnel shall not bypass proof-of-control requirements.
- **SEC-012** Authentication endpoints shall apply per-account and per-source throttling with progressive delay or equivalent defenses. Controls shall not enable an attacker to permanently lock out a victim; suspicious attempts and recovery changes shall generate security events and risk-based alerts.
- **SEC-013** OIDC accounts shall be keyed by the validated issuer and subject identifiers, not email alone. Account linking requires an authenticated session plus fresh proof of control of both identities; provider callbacks validate state, nonce, issuer, audience, signature, redirect URI, and authorization-code binding.
- **SEC-014** Privileged and high-impact actions—including role/trust changes, private-address export or bulk access, identity-proofing decisions, credential/MFA changes, ticket-domain allowlist changes, and destructive policy changes—require recent step-up authentication. Self-approval and self-escalation are prohibited; critical administrator actions require an independently authorized second approver where technically supported, otherwise the action is blocked pending review.
- **SEC-015** Every externally initiated mutation shall support replay-safe processing through idempotency keys, event/version preconditions, unique constraints, or an equivalent mechanism. Duplicate, reordered, stale, or concurrent requests shall not repeat notifications, consume capacity twice, restore revoked access, or overwrite newer state.
- **SEC-016** Third-party callbacks and webhooks shall authenticate the sender, verify signatures over the raw request, enforce timestamp/replay windows, validate schema and tenant/context binding, and process idempotently. A callback shall never directly grant a role, release a location, or complete identity proofing without server-side policy evaluation.
- **SEC-017** Security-relevant secrets and encryption keys shall have documented ownership, least-privilege access, rotation, revocation, versioning, and recovery procedures. Private-location encryption shall use authenticated encryption with record/context binding; plaintext and keys shall never coexist in logs, analytics, caches, or backups beyond the minimum required processing boundary.
- **SEC-018** Abuse-prone and computationally expensive operations shall have actor, IP/network, object, and system-wide quotas, bounded request/body/decompression sizes, pagination ceilings, timeouts, concurrency limits, and backpressure. The system shall degrade safely under dependency failure without bypassing authorization, privacy, capacity, or moderation holds.
- **SEC-019** Production shall enforce an explicit cross-origin and browser security policy: allowlisted origins, no credentialed wildcard CORS, anti-clickjacking/frame restrictions, restrictive content security policy, trusted-host validation, and cache controls that prevent authenticated or restricted responses from being stored in shared caches.
- **SEC-020** Security controls and configuration shall fail closed. Authorization, key management, identity proofing, audit, and policy-service outages shall not cause privileged access, publication, RSVP confirmation, or location release; explicitly documented read-only public functionality may remain available.

## 21. Audit (AUDIT)

- **AUDIT-001** Audited: organizer approvals/suspensions, publication, material edits, cancellations, visibility changes, private-location access, capacity changes, co-organizer permission changes, moderator and admin actions, fraud determinations, duplicate merges, attendee removals, sensitive exports.
- **AUDIT-002** Each entry: actor, action, target, timestamp, result, old/new values, reason where required, correlation ID. Logs are tamper-protected, access-restricted, and record references or redacted values where sufficient.
- **AUDIT-003** A sensitive mutation and its required audit record shall commit atomically or through a durable transactional outbox with detectable reconciliation. If durable audit capture cannot be guaranteed, the mutation fails closed. Audit access, export, retention changes, and deletion attempts are themselves audited; clocks are synchronized and actor identity distinguishes user, service, and impersonation-free support activity.

## 22. Privacy (PRIV)

- **PRIV-001** Data minimization: collect only what defined functions require. Publish clear notices covering what, why, who, retention, and deletion requests.
- **PRIV-002** Organizer verification data is never public by default. Precise location data is never sold. Private-event addresses are never used for advertising or profiling. Users control non-essential notifications.
- **PRIV-003** Data retention, backup retention, legal holds, access requests, correction, and deletion shall be defined by data class. Deletion shall cover replicas, search indexes, caches, derived data, and backups according to a documented schedule; legal holds are authorized, scoped, expiring, and audited. Restores shall reapply deletions and current authorization state before restored data becomes available.
- **PRIV-004** Bulk exports and administrative searches of identity, attendee, report, verification, or restricted-location data require a stated purpose, least-privilege scope, step-up authentication, audit logging, bounded result size, and non-shareable delivery. Spreadsheet exports neutralize formula injection and omit fields not explicitly authorized.

## 23. Accessibility (A11Y)

- **A11Y-001** WCAG 2.2 AA target; keyboard-accessible core workflows; accessible names on controls; no color-only information; alt text on event images.
- **A11Y-002** Structured accessibility fields: wheelchair access, accessible restrooms, seating, sign-language interpretation, captioning, sensory considerations, transit access. "Verified" accessibility claims require a defined verification process.

## 24. Performance and Reliability (PERF, REL)

- **PERF-001** Public search responds within 2 seconds under normal load; large result sets paginated; capacity enforcement remains correct under concurrency.
- **REL-001** Material changes commit before notifications; notification failure never rolls back a valid change; failed deliveries retried under a bounded policy; event/RSVP consistency preserved through partial failures.

## 25. Acceptance Tests

```gherkin
Scenario: AT-001 Unapproved organizer cannot publish
  Given a user with organizer status "Applicant" has a draft event
  When the user attempts to publish
  Then publication is denied, the event remains a draft, and the denial is logged

Scenario: AT-002 Private address is not public
  Given a private-residence event with visibility "After approved RSVP"
  When an unauthenticated visitor views the event
  Then no exact address or coordinates are returned and only the approved approximate location is displayed

Scenario: AT-003 Material change requires notification
  Given a published event with confirmed attendees
  When an authorized organizer changes the venue
  Then a new revision is created, the previous venue is preserved, attendees are notified, and a cancellation option is provided

Scenario: AT-004 Cancellation stops RSVPs
  Given a canceled event
  When an attendee attempts to RSVP
  Then the RSVP is rejected and no capacity is reserved

Scenario: AT-005 Co-organizer permissions are scoped
  Given a user is a co-organizer on Event A with no role on Event B
  When the user attempts to edit Event B
  Then access is denied and Event B is unchanged

Scenario: AT-006 Duplicate warning
  Given a published event at the same venue and time
  When an organizer creates a substantially similar event
  Then possible duplicates are displayed, draft-save is allowed, and publication requires resolution or review

Scenario: AT-007 Capacity is enforced atomically
  Given an event with one remaining place
  When two attendees submit concurrent RSVPs
  Then exactly one is confirmed, the other is rejected or waitlisted, and confirmed attendance does not exceed capacity

Scenario: AT-008 Moderator cannot silently rewrite content
  Given inaccurate but non-dangerous event content
  When a moderator reviews it
  Then the moderator may request a correction or restrict publication but may not silently replace the description

Scenario: AT-009 Hidden location absent from public API
  Given an event with a restricted exact address and an unauthorized requester
  When the event is retrieved via the public API
  Then the response contains no exact address, coordinates, or address-revealing identifier

Scenario: AT-010 Canceled event remains auditable
  Given a published event with attendees
  When an authorized organizer cancels it
  Then status becomes "Canceled", the record is preserved, attendees are notified, and the reason is recorded

Scenario: AT-011 Address release re-checks authorization
  Given an attendee whose RSVP was revoked after approval
  When the timed address release executes
  Then the address is not released to that attendee and the block is logged

Scenario: AT-012 Non-allowlisted ticket link blocked
  Given an organizer submits an external ticket URL on a non-allowlisted domain
  When the event is submitted for publication
  Then publication is held for moderator approval of the link

Scenario: AT-013 Recovery cannot be used for account takeover
  Given an account protected by MFA and active sessions
  When recovery succeeds using a valid single-use token
  Then the token cannot be replayed, affected sessions are invalidated, the user is notified independently, and the event is audited

Scenario: AT-014 Sensitive mutation fails without durable audit
  Given the audit sink and transactional outbox cannot durably accept a record
  When an administrator attempts a role elevation
  Then the role is unchanged, the request fails closed, and an operational alert is raised

Scenario: AT-015 Signed callback cannot be replayed
  Given a valid identity-provider callback has already been processed
  When the same signed callback is submitted again
  Then no state transition is repeated and the replay is recorded as a security event

Scenario: AT-016 Shared caches cannot disclose a restricted location
  Given an authorized attendee retrieves a private event address
  When an unauthorized requester uses the same URL through any shared cache
  Then the restricted response is not served and no sensitive value is present in cache metadata
```

## 26. Resolved Product Decisions

| Question | Decision |
|---|---|
| Verification provider | Vendor-agnostic; NIST SP 800-63A IAL2 equivalence required (gov ID + liveness) |
| Private-residence max capacity | Limited 10 / Approved 25 / Verified 50; above tier cap needs moderator approval |
| Limited-trust manual review | First 3 events, all private-residence, paid/payment-link events, capacity > 50, edits within 24h of start |
| Enhanced-moderation categories | Minors-focused, alcohol/cannabis, late-night (22:00–06:00), health-claim events, gatherings > 250, all private residences |
| Attendee approval for private residences | Mandatory for every private-residence event |
| Verification record retention | Result: account life + 1 year. Raw ID documents: deleted within 30 days of decision |
| External ticket links | Admin-managed domain allowlist; exceptions need moderator approval |
| Notification channels | Email (required) and optional push at launch; SMS deferred |
| Verified-badge evidence | IAL2 proofing + 3 clean completed events + venue/business confirmation |
| Appeal timeline | Acknowledge 2 business days; decide 7; suspensions 3; second reviewer |
