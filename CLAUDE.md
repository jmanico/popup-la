# CLAUDE.md — PopUp-LA

Mobile + API platform for discovering and publishing temporary events in Los Angeles.
**Bootstrap mode:** requirements, architecture, security, and design are defined; no
implementation exists yet. Do not invent workflow, tooling, or infrastructure that these
documents leave undecided — mark it `TO BE DECIDED` and ask.

## Canonical sources (read before acting)

@REQUIREMENTS.md
@ARCHITECTURE.md
@SECURITY.md
@DESIGN.md

These four files are authoritative and non-overlapping: **what** the system must do
(REQUIREMENTS), **how** it is structured (ARCHITECTURE), **security guardrails** (SECURITY),
**visual language** (DESIGN). This file adds only working style and cross-cutting behavior;
it does not restate rules already in those files. On any conflict, the linked document wins —
fix the conflict there, not by contradicting it here.

## Guardrails (non-negotiable)

- **Server is authoritative; clients are untrusted.** This is the root invariant behind
  authorization, capacity, and location release. See AUTHZ / EVT-003 / CAP-002 / LOC-006.
- **Never expose restricted location data** across any boundary toward an unauthorized
  consumer (LOC-004, SEARCH-002). When touching events, RSVP, search, media, notifications,
  or exports, assume a private address is in scope and prove it cannot leak.
- **Honest UI, always.** Approval, safety, refunds, and occupancy are never presented as
  guaranteed (ORG-008, CANCEL-003, CAP-005, DESIGN §6).
- **Every sensitive action writes an audit entry** (AUDIT-001/002). No sensitive mutation
  ships without its audit write.
- **Secure-by-default coding** per SECURITY.md — parameterized queries, contextual output
  encoding, deny-by-default authz, no secrets in source, rate limits on abuse-prone endpoints.
- **Dependencies:** prefer zero new dependencies; justify any addition per SECURITY.md
  Dependency Rules. Do not add one when first-party code will do.

## Working style

- When a mechanism, vendor, or scope is undecided in the source docs (notably the
  native-vs-web client conflict, data store, hosting, and IAL2 vendor), treat it as
  `TO BE DECIDED` — do not pick one silently.
- Prefer small, individually testable units; security logic (authz, crypto, validation)
  lives in dedicated modules with tests (SECURITY.md code-quality rule).
- Every new feature traces to a requirement ID. If none exists, a requirement must be
  written first (see below) — do not implement un-specified behavior.

## GitHub issues — REQUIRED format

Every new GitHub issue MUST follow **@REQUIREMENT_TEMPLATE.md** so each issue is a
structured, testable requirement (metadata, RFC-2119 description, security context,
standards alignment, testable acceptance criteria, failure behavior). Issues that are not
expressible as a requirement (pure chores, questions) should say so explicitly and skip
the sections that do not apply — but feature and security work uses the full template.

## Undecided workflow — placeholders

No project conventions are defined yet for these. Ask the user before assuming any:

- **Build / run / test commands:** `TO BE DECIDED`
- **Lint / format / typecheck:** `TO BE DECIDED`
- **Branch, commit, and PR conventions:** `TO BE DECIDED`
- **CI / security gates (SAST, SCA, IaC scan):** `TO BE DECIDED`
- **Directory layout for API and client code:** `TO BE DECIDED`
