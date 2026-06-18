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
- React single-page app (browser) → REST API (Flask) → MySQL, with Redis as a supporting store (server-side sessions and rate-limit counters). The API tier holds no per-instance session state (session data lives in Redis), so it scales horizontally behind TLS termination. Standings are computed on demand for v0.2 (see Data); a cache is a deferred optimization (NFR-1).
- The Flask API is the sole enforcement point for authorization, state transitions, and validation. The React client is untrusted; client-side checks are UX only (SR-5).

### Server (Flask, API-centric)
- Layering: HTTP/route layer (request parsing, authz guards, CSRF) → service/domain layer (state machine, ranking, validation grammar) → repository/data layer (MySQL). Business rules live in the domain layer so they are testable independent of HTTP.
- Match state machine (Section 6) implemented as an explicit transition table; illegal transitions rejected server-side, never silent no-ops (SR-10). Guards (C1–C3, BR-5, BR-9, tournament-active, `confirmation_source`) evaluated in the domain layer.
- Result validation grammar (Section 7) and ranking cascade (Section 8) implemented as pure functions with deterministic output; `winner_id` and standings are always derived, never client-supplied (FR-20, FR-27, BR-6).
- Time-based transitions (proposal expiry FR-15, result auto-confirm FR-19) and tournament status transitions (FR-10): DECIDED — **on-access lazy evaluation only**, no scheduled worker. Every read/action path that depends on a time-driven state (tournament status checks, match/result access, dispute queue, standings/scoreboard computation) first evaluates and applies any due transition so the observable behavior in FR-10/FR-15/FR-19 holds without a background job. Consequence to honor: an entity that is never read does not transition on its own; the read points above must therefore cover every place a stale state would be user-visible or standings-affecting. All time logic in UTC (NFR-2). Lazily-applied system transitions use the reserved system actor id in the audit log (Section 4.8).
- Audit log: append-only write on every score-affecting action; no update/delete path exposed to any role (SR-11, BR-8). DECIDED — enforcement is **DB GRANTs + app discipline**: the application's MySQL user is granted only INSERT and SELECT on `audit_log` (no UPDATE/DELETE/ALTER), and no service/repository code exposes a mutation path. Schema migrations use a separate privileged credential.

### Client (React)
- Public, unauthenticated scoreboard view (active tournament + historical selector, FR-28, FR-30). Authenticated views for profile, enrollment, matchmaking, results, disputes; admin views for tournaments, dispute queue, account actions.
- All privileged actions gated server-side; the client renders capability based on role but never relies on it for security.

### Authentication & session
- Auth subsystem covers player email/password login plus email verification and password reset (FR-1..FR-5), and admin passkey/WebAuthn (FR-6); session and CSRF handling are cross-cutting. Control specifics (password hashing, token hygiene, cookie flags, session lifecycle, anti-CSRF) are owned by SECURITY.md per SR-1, SR-3, SR-4, SR-14, SR-18 — not restated here.
- Architectural note: `password_hash` is nullable to support passkey-only admin accounts (REQUIREMENTS.md §4.1); player password and admin WebAuthn credentials are modeled as separate records. Admin invitation is modeled as a single-use token record (issuer, intended email, authorized role, expiry, consumed-at) whose issuance is audited as `account.invite`; token hygiene and session-id regeneration on auth/privilege-change are session-subsystem behaviors owned by SECURITY.md (SR-3, SR-4, SR-11).
- DECIDED — library choices (see Dependency Rules → Selected dependencies): admin WebAuthn via **py_webauthn** (`webauthn`), with each passkey stored as its own credential record (`credential_id`, `public_key`, `sign_count`, `transports`, `aaguid`, `created_at`) separate from the user row; server-side sessions via **Flask-Session** backed by **Redis**; anti-CSRF via **Flask-WTF** (`CSRFProtect`, synchronizer token echoed in an `X-CSRFToken` header). Token/cookie hygiene and lifecycle remain owned by SECURITY.md.

### Data (MySQL, 3NF)
- Entities map directly to REQUIREMENTS.md Section 4: User, Profile (1:1), Tournament, Enrollment, Match, Result (1:1 active per match), Dispute, Audit Log. Normalize to 3NF. DECIDED — a Result's set scores are stored in a normalized child table **`result_sets`** (`result_id`, `set_index`, `games_proposer`, `games_opponent`; PK/unique on (`result_id`, `set_index`)), not a JSON column — keeping the model in 3NF, integer-validated at the DB, and directly aggregable for the §8 set/game differentials.
- Database-enforced invariants (not app-only):
  - C1 no self-match; C2 both enrolled at acceptance.
  - C4 non-overlapping tournament windows: DECIDED — MySQL has no range-exclusion constraint, so C4 is enforced in the service layer inside a **SERIALIZABLE** transaction (overlap SELECT then insert/update) plus an application guard, rather than declaratively. Documented limitation; contention is negligible because tournaments are sequential and rare (RD-1). (C4, FR-9, RD-1.)
  - C3 one official match per pair per tournament: DECIDED — enforced via a **STORED generated nullable key column** (e.g. `pairing_key` = concat of `tournament_id`/`pair_low`/`pair_high` when `status` is an occupying status, else NULL) with a plain **UNIQUE** index over it; MySQL ignores NULLs, so the uniqueness applies only over occupying statuses. Pure-declarative and race-safe, no triggers (SR-9, BR-3).
  - Enrollment unique on (`user_id`, `tournament_id`).
  - `pair_low`/`pair_high` as generated columns from proposer/opponent ids for order-independent uniqueness.
- Standings: computed from confirmed + resolved (non-voided) matches only. DECIDED — v0.2 **computes standings on demand** from the §8 pure function (correct and adequate at hundreds of users); a cache/materialized view with bounded staleness < 60s is a deferred optimization behind a documented seam, revisited only if read load warrants it (NFR-1, FR-27, BR-4).

### Cross-cutting
- Security controls that span all layers — server-side input validation/grammar, request-binding field allowlists (mass-assignment guard at the route→service boundary), output encoding + CSP, TLS with HTTP→HTTPS redirect and HSTS, uniform non-leaking errors, and separation of duties on admin adjudication — are implemented per SR-12, SR-19, SR-13, SR-15, SR-16, SR-17 (BR-9; operationally ≥2 admins, NFR-5). Control detail is owned by SECURITY.md and not restated here.

### Deployment (terraform)
- The v0.2 deliverable is a single hardened Docker image (non-root, minimal pinned base, no build tools in the runtime layer); full Terraform topology is out of v0.2 scope. Concrete topology (compute, managed MySQL, managed Redis, TLS/HSTS termination) remains TO BE DECIDED; scale target is hundreds of users (single small app tier + one managed DB + one small Redis is a reasonable ASSUMPTION). No scheduled-worker component is provisioned — time transitions are lazy (see Server).
- Secrets and signing keys are accessed through an application-level abstraction (a `SecretProvider`/`KeyProvider`-style interface) with an environment-variable-backed implementation now, designed so an AWS KMS / Secrets Manager (or similar) implementation can be slotted in later without touching call sites. The concrete key-management backend is TO BE DECIDED with deployment.

### Selected technology decisions (v0.2, resolved)
- Testing: backend **pytest** (+`pytest-cov`), frontend **Vitest** + React Testing Library; CI gate **≥80% branch coverage**.
- Lint/format/types: backend **Ruff** + **Black**; frontend **ESLint** + **Prettier** + **TypeScript** (strict); enforced via the **`pre-commit`** framework.
- CI/CD: **GitHub Actions** — runs tests, **pip-audit** + **npm audit** (dependency CVEs) and **gitleaks** (secret scan) on every PR; merge blocks on known unpatched CVEs.
- Git workflow: trunk-based with short-lived branches; PRs require 1 approving review + green CI; **squash-merge**; **Conventional Commits**.
- Session/CSRF/rate-limit: **Flask-Session + Redis**; **Flask-WTF** CSRF; **Flask-Limiter + Redis** (oversized-body via Flask `MAX_CONTENT_LENGTH`).
- WebAuthn: **py_webauthn**. Email: provider-agnostic SMTP `EmailSender` abstraction (concrete provider chosen at deploy).
- C3: generated nullable key column + UNIQUE index. C4: SERIALIZABLE tx + service guard. Audit immutability: DB GRANTs (INSERT/SELECT only) + app discipline. Time transitions: on-access lazy only. `result_sets` child table. Standings computed on demand.

### Explicit unknowns (do not guess)
- Email delivery provider (the `EmailSender` SMTP backend) and its credentials: TO BE DECIDED at deploy.
- Key-management / secret-store backend behind the `SecretProvider` abstraction (e.g. AWS KMS): TO BE DECIDED.
- Terraform topology / hosting / data-region (and GDPR retention-vs-erasure policy, SECURITY.md): TO BE DECIDED — only the hardened Docker image is in v0.2 scope.

## Requirement Traceability

| Architecture component / boundary | Requirement groups | Notes |
|---|---|---|
| React SPA + Flask REST API boundary | Section 3 actors; SR-5 deny-by-default | Server is sole authz enforcement point |
| Auth/session subsystem (player password, admin passkey, tokens) | FR-1..FR-6; SR-1..SR-4, SR-18 | Argon2id; passkey-only admins; invite + seeded first admin; session-id regeneration (SR-3); single-use bound invite token (SR-4) |
| Request-binding / mass-assignment guard | SR-19; FR-1, FR-7, FR-17 | Per-endpoint writable-field allowlist; `role`/`status`/`winner_id`/`confirmation_source` never client-set |
| Session/CSRF/transport hardening | SR-3, SR-14, SR-15, SR-16 | Cookie flags, anti-CSRF, TLS/HSTS, uniform errors |
| Authorization layer (object- + function-level, separation of duties) | SR-5, SR-6, SR-7, SR-17; BR-9; NFR-5 | BOLA/IDOR is highest risk; SoD needs ≥2 admins |
| Profile module | FR-7, FR-8 | Object-level authz; non-trivial display_name |
| Tournament lifecycle + lazy evaluation | FR-9, FR-10; C4; RD-1 | On-access lazy transitions (no scheduler); C4 via SERIALIZABLE tx + service guard |
| Enrollment module | FR-11, FR-12; C2 (BR-2) | Unique (user, tournament) |
| Match state machine + guards | FR-13..FR-16, FR-22; Section 6; C1, C3; SR-10; BR-1, BR-3 | Explicit transition table; C3 via generated nullable key column + UNIQUE index |
| Result entry/approval/auto-confirm + `confirmation_source` | FR-17..FR-21; RD-5; BR-5, BR-7, BR-10; SR-8 | Auto-confirm escalation path |
| Result validation grammar | Section 7; FR-17, FR-24; SR-12 | Pure, deterministic; applied at submit and resolve |
| Dispute + admin resolution/void | FR-22..FR-26; SR-7, SR-17; BR-9 | Admin-only queue; SoD enforced |
| Ranking + standings engine | Section 8; FR-27; BR-4, BR-6 | Derived, total, deterministic ordering |
| Public scoreboard + history | FR-28, FR-29, FR-30; NFR-1, NFR-3 | Computed on demand (caching deferred); no private data exposed |
| Audit log subsystem | Section 4.8; SR-11; BR-8; NFR-4 | Append-only; score actions + suspend/reactivate/`account.invite`; immutability via DB GRANTs (INSERT/SELECT only) + app discipline |
| MySQL schema (3NF) + DB constraints | Section 4; C1–C4; SR-9 | C3 = generated nullable key + UNIQUE; C4 = SERIALIZABLE tx; `result_sets` child table |
| Email subsystem (verification, reset) | FR-2, FR-5; SR-4 | Provider-agnostic SMTP `EmailSender`; concrete provider `TO BE DECIDED` |
| Session / CSRF / rate-limit (Redis) | SR-2, SR-3, SR-14 | Flask-Session + Redis; Flask-WTF CSRF; Flask-Limiter + Redis |
| Terraform deployment | Deployment input; NFR-1, NFR-2 | Hardened Docker image in scope; full topology `TO BE DECIDED` |

## Dependency Rules

- Do not add a dependency when the standard library or a few lines of first-party code will do.
- Prefer zero new dependencies. If a library is required, justify it in the PR description.
- Only use libraries that are actively maintained (commit or release within the last 12 months).
- Only use the latest stable major version. No deprecated, abandoned, or pre-release packages.
- Reject any library with known unpatched CVEs. Check before adding and on every update.
- Audit transitive dependencies, not just direct ones. A small direct dep with a large or unvetted tree is a rejection.
- Pin exact versions with a committed lockfile. No floating ranges in production.
- Prefer libraries with a narrow scope, minimal dependencies of their own, and a clear security track record.

### Selected dependencies (v0.2)

Chosen under the rules above; each must be pinned in the committed lockfile and re-checked for CVEs on update. Justify any addition beyond this list in its PR.

| Concern | Dependency | Notes |
|---|---|---|
| Password hashing (SR-1) | `argon2-cffi` (Argon2id) | Memory-hard, salted |
| WebAuthn / passkeys (FR-6, SR-18) | `webauthn` (py_webauthn) | Relying-party verification; per-passkey credential records |
| Server-side sessions (SR-3) | `Flask-Session` + Redis | Opaque cookie id; state in Redis; enables forced invalidation/regeneration |
| Anti-CSRF (SR-14) | `Flask-WTF` (`CSRFProtect`) | Synchronizer token via `X-CSRFToken` |
| Rate limiting (SR-2) | `Flask-Limiter` + Redis | Per-endpoint limits; oversized body via Flask `MAX_CONTENT_LENGTH` |
| Email (FR-2, FR-5) | stdlib `smtplib` behind an `EmailSender` interface | Provider-agnostic; concrete SMTP provider chosen at deploy |
| Secrets/keys | first-party `SecretProvider` abstraction (env-backed) | Swappable to AWS KMS / Secrets Manager later |
| Backend tests | `pytest` + `pytest-cov` | ≥80% branch gate |
| Frontend tests | `Vitest` + React Testing Library | ≥80% branch gate |
| Lint/format (py) | `Ruff` + `Black` | via `pre-commit` |
| Lint/format (js) | `ESLint` + `Prettier` + `TypeScript` | via `pre-commit` |

ORM/DB driver and migration tool (e.g. SQLAlchemy + Alembic vs. a lighter approach) are selected in the schema-foundation issues under these same rules.
