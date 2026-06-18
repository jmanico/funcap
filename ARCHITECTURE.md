# Funcap Architecture

Status: Bootstrap / Provisional
Derived from: REQUIREMENTS.md (Draft v0.2). REQUIREMENTS.md is the source of truth; where this document and REQUIREMENTS.md disagree, REQUIREMENTS.md wins. This repository currently contains only REQUIREMENTS.md; nothing below is reverse-engineered from code.

## Required Architecture Inputs

- `Requirements source: REQUIREMENTS.md`
- `System purpose: see REQUIREMENTS.md §1 (recurring community tennis tournament series; trust backed by mutual confirmation, an immutable audit log, and an admin dispute backstop).`
- `Primary use cases: player onboarding, tournament lifecycle, enrollment, matchmaking, result entry/confirmation, disputes and admin resolution, standings/scoreboard with history — see REQUIREMENTS.md §5 (FR-1..FR-30).`
- `Target users / actors: Anonymous visitor, Player, Admin — see REQUIREMENTS.md §3 (admin is a superset of player; deny-by-default authorization).`
- `Runtime environment: web application`
- `Server framework: Python Flask, API centric`
- `Client framework: ReactJS`
- `API style and integration model: REST`
- `Authentication and session model: Normal user (username/password); Admin (passkey / WebAuthn)`
- `Data model expectations: MySQL - Third Normal Form`
- `Deployment model: terraform`
- `Scale expectations: hundreds of users`
- `Security expectations: admin security and tournament security to be solid`

## Initial Architecture (Provisional)

First-pass design that satisfies REQUIREMENTS.md. Concrete choices below come only from the explicit inputs above or from REQUIREMENTS.md; everything else is labeled ASSUMPTION or TO BE DECIDED.

### Shape
- React single-page app (browser) → REST API (Flask) → MySQL. Stateless API tier behind TLS termination; standings reads may be served from a cache/materialized view (NFR-1).
- The Flask API is the sole enforcement point for authorization, state transitions, and validation. The React client is untrusted; client-side checks are UX only (SR-5).

### Server (Flask, API-centric)
- Layering: HTTP/route layer (request parsing, authz guards, CSRF) → service/domain layer (state machine, ranking, validation grammar) → repository/data layer (MySQL). Business rules live in the domain layer so they are testable independent of HTTP.
- Match state machine (Section 6) implemented as an explicit transition table; illegal transitions rejected server-side, never silent no-ops (SR-10). Guards (C1–C3, BR-5, BR-9, tournament-active, `confirmation_source`) evaluated in the domain layer.
- Result validation grammar (Section 7) and ranking cascade (Section 8) implemented as pure functions with deterministic output; `winner_id` and standings are always derived, never client-supplied (FR-20, FR-27, BR-6).
- Time-based transitions (proposal expiry FR-15, result auto-confirm FR-19) and tournament status transitions (FR-10): a scheduled worker is the ASSUMPTION; on-access lazy evaluation is an allowed alternative per REQUIREMENTS.md. Mechanism is TO BE DECIDED. All time logic in UTC (NFR-2). System-initiated transitions use the reserved system actor id in the audit log (Section 4.8).
- Audit log: append-only write on every score-affecting action; no update/delete path exposed to any role (SR-11, BR-8). Enforcement mechanism (app-level discipline vs. DB-level revoke of UPDATE/DELETE) is TO BE DECIDED.

### Client (React)
- Public, unauthenticated scoreboard view (active tournament + historical selector, FR-28, FR-30). Authenticated views for profile, enrollment, matchmaking, results, disputes; admin views for tournaments, dispute queue, account actions.
- All privileged actions gated server-side; the client renders capability based on role but never relies on it for security.

### Authentication & session
- Auth subsystem covers player email/password login plus email verification and password reset (FR-1..FR-5), and admin passkey/WebAuthn (FR-6); session and CSRF handling are cross-cutting. Control specifics (password hashing, token hygiene, cookie flags, session lifecycle, anti-CSRF) are owned by SECURITY.md per SR-1, SR-3, SR-4, SR-14, SR-18 — not restated here.
- Architectural note: `password_hash` is nullable to support passkey-only admin accounts (REQUIREMENTS.md §4.1); player password and admin WebAuthn credentials are modeled as separate records. Admin invitation is modeled as a single-use token record (issuer, intended email, authorized role, expiry, consumed-at) whose issuance is audited as `account.invite`; token hygiene and session-id regeneration on auth/privilege-change are session-subsystem behaviors owned by SECURITY.md (SR-3, SR-4, SR-11).
- ASSUMPTION: a single WebAuthn relying-party library and a session/cookie library will be needed; specific libraries are TO BE DECIDED under the Dependency Rules.

### Data (MySQL, 3NF)
- Entities map directly to REQUIREMENTS.md Section 4: User, Profile (1:1), Tournament, Enrollment, Match, Result (1:1 active per match), Dispute, Audit Log. Normalize to 3NF.
- Database-enforced invariants (not app-only):
  - C1 no self-match; C2 both enrolled at acceptance; C4 non-overlapping tournament windows.
  - C3 one official match per pair per tournament: partial/filtered unique index on (`tournament_id`, `pair_low`, `pair_high`) over occupying statuses. NOTE/RISK: MySQL lacks native partial indexes; enforcement approach (trigger, generated/nullable key column, or upsert-guard) is TO BE DECIDED (SR-9, BR-3).
  - Enrollment unique on (`user_id`, `tournament_id`).
  - `pair_low`/`pair_high` as generated columns from proposer/opponent ids for order-independent uniqueness.
- Standings: computed from confirmed + resolved (non-voided) matches only; may be cached/materialized with bounded staleness < 60s (NFR-1, FR-27, BR-4).

### Cross-cutting
- Security controls that span all layers — server-side input validation/grammar, request-binding field allowlists (mass-assignment guard at the route→service boundary), output encoding + CSP, TLS with HTTP→HTTPS redirect and HSTS, uniform non-leaking errors, and separation of duties on admin adjudication — are implemented per SR-12, SR-19, SR-13, SR-15, SR-16, SR-17 (BR-9; operationally ≥2 admins, NFR-5). Control detail is owned by SECURITY.md and not restated here.

### Deployment (terraform)
- Infrastructure provisioned via Terraform. Concrete topology (compute, managed MySQL, secrets, scheduler, TLS/HSTS termination, cache) is TO BE DECIDED; scale target is hundreds of users (single small app tier + one managed DB is a reasonable ASSUMPTION).

### Explicit unknowns (do not guess)
- Time-transition mechanism (scheduler vs. lazy): TO BE DECIDED.
- C3 enforcement technique on MySQL: TO BE DECIDED.
- Audit-log immutability enforcement layer: TO BE DECIDED.
- Email delivery provider, WebAuthn library, session/CSRF libraries, caching layer, hosting topology: TO BE DECIDED.

## Requirement Traceability

| Architecture component / boundary | Requirement groups | Notes |
|---|---|---|
| React SPA + Flask REST API boundary | Section 3 actors; SR-5 deny-by-default | Server is sole authz enforcement point |
| Auth/session subsystem (player password, admin passkey, tokens) | FR-1..FR-6; SR-1..SR-4, SR-18 | Argon2id; passkey-only admins; invite + seeded first admin; session-id regeneration (SR-3); single-use bound invite token (SR-4) |
| Request-binding / mass-assignment guard | SR-19; FR-1, FR-7, FR-17 | Per-endpoint writable-field allowlist; `role`/`status`/`winner_id`/`confirmation_source` never client-set |
| Session/CSRF/transport hardening | SR-3, SR-14, SR-15, SR-16 | Cookie flags, anti-CSRF, TLS/HSTS, uniform errors |
| Authorization layer (object- + function-level, separation of duties) | SR-5, SR-6, SR-7, SR-17; BR-9; NFR-5 | BOLA/IDOR is highest risk; SoD needs ≥2 admins |
| Profile module | FR-7, FR-8 | Object-level authz; non-trivial display_name |
| Tournament lifecycle + scheduler/lazy evaluation | FR-9, FR-10; C4; RD-1 | Transition mechanism `TO BE DECIDED` |
| Enrollment module | FR-11, FR-12; C2 (BR-2) | Unique (user, tournament) |
| Match state machine + guards | FR-13..FR-16, FR-22; Section 6; C1, C3; SR-10; BR-1, BR-3 | Explicit transition table; C3 DB enforcement `TO BE DECIDED` |
| Result entry/approval/auto-confirm + `confirmation_source` | FR-17..FR-21; RD-5; BR-5, BR-7, BR-10; SR-8 | Auto-confirm escalation path |
| Result validation grammar | Section 7; FR-17, FR-24; SR-12 | Pure, deterministic; applied at submit and resolve |
| Dispute + admin resolution/void | FR-22..FR-26; SR-7, SR-17; BR-9 | Admin-only queue; SoD enforced |
| Ranking + standings engine | Section 8; FR-27; BR-4, BR-6 | Derived, total, deterministic ordering |
| Public scoreboard + history | FR-28, FR-29, FR-30; NFR-1, NFR-3 | Cache/materialize; no private data exposed |
| Audit log subsystem | Section 4.8; SR-11; BR-8; NFR-4 | Append-only; score actions + suspend/reactivate/`account.invite`; immutability layer `TO BE DECIDED` |
| MySQL schema (3NF) + DB constraints | Section 4; C1–C4; SR-9 | C3 technique on MySQL `TO BE DECIDED` |
| Email subsystem (verification, reset) | FR-2, FR-5; SR-4 | Provider `TO BE DECIDED` |
| Terraform deployment | Deployment input; NFR-1, NFR-2 | Topology `TO BE DECIDED` |

## Dependency Rules

- Do not add a dependency when the standard library or a few lines of first-party code will do.
- Prefer zero new dependencies. If a library is required, justify it in the PR description.
- Only use libraries that are actively maintained (commit or release within the last 12 months).
- Only use the latest stable major version. No deprecated, abandoned, or pre-release packages.
- Reject any library with known unpatched CVEs. Check before adding and on every update.
- Audit transitive dependencies, not just direct ones. A small direct dep with a large or unvetted tree is a rejection.
- Pin exact versions with a committed lockfile. No floating ranges in production.
- Prefer libraries with a narrow scope, minimal dependencies of their own, and a clear security track record.
