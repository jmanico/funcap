# Funcap Security

Status: Provisional (early-stage). Derived from REQUIREMENTS.md (Section 9 SR-1..SR-19, Section 10 BR-1..BR-10, NFR-3/4/5) and ARCHITECTURE.md. REQUIREMENTS.md is the source of truth; on conflict it wins. Rules are compressed for Claude Code — they are directives, not prose. Every rule traces to an SR/BR/NFR id or an architecture boundary.

Threat priority for this app: broken object-level authorization (BOLA/IDOR) is the highest risk, because standings derive from match/result/dispute objects (SR-6). Second: business-logic abuse (self-approval, collusion, illegal transitions). Treat both as first-class.

## STRIDE Threat Model

Bootstrap STRIDE threat model (REQ-SEC-001) over the React client ↔ Flask API trust boundary (the API is the sole enforcement point, SR-5) and the app ↔ MySQL / email / WebAuthn boundaries. Each category maps the app-specific threats to the controls that mitigate them. Controls are authored as SR/BR ids in REQUIREMENTS.md §9–§10; this table is the threat-to-control map, not a second source of truth.

| STRIDE category | App-specific threats | Primary controls |
|---|---|---|
| Spoofing | Credential stuffing/brute force; session fixation; stolen reset/verify/invite token; admin impersonation | SR-1 (Argon2id), SR-2 (throttling), SR-3 (cookie flags + session-id regeneration), SR-4 (single-use bound tokens), SR-18 (passkey-only admins) |
| Tampering | Mass-assignment of `role`/`status`/`winner_id`/`confirmation_source`; client-supplied standings; duplicate match per pair; illegal state transition | SR-19 (field allowlist), SR-12 (grammar), FR-20/BR-6 (server-derived winner/standings), SR-9/C3/BR-3 (DB-enforced pairing), SR-10 (transition guard) |
| Repudiation | Denying a result submission/approval; denying an admin adjudication or admin-invite issuance | SR-11 + §4.8 append-only audit log (score actions + suspend/reactivate/`account.invite`), BR-8 immutability |
| Information Disclosure | BOLA/IDOR on match/result/dispute objects; account enumeration; PII/email leak on scoreboard; verbose errors | SR-6 (object-level authz), SR-16 (uniform login/register/verify/reset), NFR-3 (email never public), SR-13 (encoding/CSP) |
| Denial of Service | Auth brute force; proposal/result/enrollment spam; oversized bodies; scoreboard read load | SR-2 (auth + abuse-prone endpoint rate limits, oversized-body rejection), NFR-1 (cached/materialized standings) |
| Elevation of Privilege | Player → admin via mass assignment or forged invite; self-approval; participant-admin adjudicating own match | SR-19, SR-4 (role-bound invite), SR-5/SR-7 (deny-by-default + function-level), SR-8/BR-5 (submitter≠approver), SR-17/BR-9 (separation of duties) |

## Required Security Inputs

Security-salient facts the threat model rests on — authored in REQUIREMENTS.md / ARCHITECTURE.md, referenced here, not re-derived (stack shape and data model live in ARCHITECTURE.md):
- The Flask API is the sole enforcement point; the React client is untrusted (SR-5).
- Players authenticate with email + password; admins are passkey/WebAuthn-only, invite-only, first admin seeded out-of-band (FR-6, SR-18). Sessions are cookie-based and server-issued (SR-3), so CSRF defense is required (SR-14).
- Personal data is minimal and email is never shown publicly (NFR-3); the audit log is append-only and retained for the series lifetime (SR-11, NFR-4).
- Deployment artifact is a hardened Docker image now; Terraform topology is TO BE DECIDED (ARCHITECTURE.md).

Inputs still required before these rules are final (TO BE DECIDED):
- Email delivery provider (verification + reset tokens, SR-4) and its secret handling.
- WebAuthn relying-party library and credential storage model.
- Session/CSRF library choice; cache/materialized-view layer for standings (NFR-1).
- C3 enforcement technique on MySQL (no native partial index) — trigger vs. generated nullable key (SR-9).
- Audit-log immutability mechanism — app discipline vs. DB-level revoke of UPDATE/DELETE (SR-11, BR-8).
- Hosting topology, secret store, CI/CD platform, and data-region for GDPR.

## Provisional Security Rules

### HTTP boundary
- TLS everywhere; redirect HTTP→HTTPS; set HSTS (SR-15).
- Set security headers on every response: CSP (default-src 'self'), `X-Content-Type-Options: nosniff`, `Referrer-Policy`, frame-deny. CSP is defense-in-depth for stored XSS (SR-13).
- CORS: deny by default; allowlist only the known SPA origin(s). No wildcard with credentials.
- Per-endpoint rate limiting; stricter throttling on auth, register, reset, verify (SR-2). Reject oversized bodies.

### Authentication & session
- Player passwords: Argon2id (bcrypt acceptable), salted, memory-hard. Never plaintext or fast/unsalted hashes (SR-1).
- Admins: WebAuthn passkeys only; no admin password path; invitations issuable only by an existing admin; first admin via seed config, never the app (FR-6, SR-18).
- Session cookies: `HttpOnly`, `Secure`, `SameSite`; idle + absolute timeout; invalidate on logout and on password change; regenerate the session id on login and on privilege-level change (anti session-fixation) (SR-3).
- Verification/reset/admin-invitation tokens: single-use, time-limited, cryptographically random, bound to the intended recipient; invalidate on use; an invite grants only its authorized role (SR-4). Uniform responses across login/register/verify/reset — never reveal whether an email is registered (SR-16).
- All state-changing requests carry an anti-CSRF token (or equivalent) for the cookie session (SR-14).

### Authorization (highest priority)
- Deny-by-default, enforced server-side on every state-changing AND data-returning endpoint. Client checks are UX only (SR-5).
- Object-level: a player may submit/approve/reject/dispute/escalate only matches they participate in; verify participant identity against `proposer_id`/`opponent_id` on every object access — never trust an id from the request as authorization (SR-6, BOLA/IDOR).
- Function-level: tournament management, dispute resolution, account suspend/reactivate, and result void are admin-only (SR-7).
- Separation of duties: the acting admin's id must differ from both participants for resolve/void/final-score; enforce server-side (SR-17, BR-9). Requires ≥2 admins operationally (NFR-5).

### Input validation & business-logic integrity
- Validate all input server-side against an allowlist/grammar: set scores per Section 7 grammar, NTRP per its scale, status enums (SR-12). Reject, do not coerce.
- Derive `winner_id` and standings server-side from validated scores — never accept them from the client (FR-20, BR-6).
- Submitter ≠ approver (SR-8, BR-5). Reject illegal state-machine transitions; never silent no-op (SR-10).
- Enforce one-official-match-per-pair-per-tournament at the DB layer, not only in app code, to survive races (SR-9, BR-3, constraint C3).
- Confirmed-by-approval and resolved results are immutable to players; auto-confirmed results mutable only via the escalation path until tournament completes (BR-7, BR-10).
- Mass-assignment guard: bind only an explicit per-endpoint allowlist of client-writable fields. Never accept `role`, `status`, `email_verified_at`, `winner_id`, `confirmation_source`, `approved_by`/`approved_at`, `submitted_by`, `pair_low`/`pair_high`, ids, or timestamps from the request body (SR-19).

### Output encoding
- Contextually output-encode user-controlled display fields (`display_name`, `bio`, `location`) wherever rendered; rely on React's default escaping and never use `dangerouslySetInnerHTML` on user data (SR-13).

### Secret handling
- No secrets in source, client bundle, or logs. Load DB creds, token-signing keys, email-provider keys, WebAuthn RP secrets from environment/secret store injected at deploy.
- Distinct secrets per environment; rotate on exposure. Lockfile-pinned dependencies (see ARCHITECTURE.md Dependency Rules) to limit supply-chain risk.

### Logging & error handling
- Error responses leak no stack traces, internal ids, or account-existence signals; uniform auth/reset responses (SR-16).
- Audit log is append-only and immutable to all roles; write every score-affecting action (submit/approve/reject/auto-confirm/dispute/resolve/void) and security-relevant account action (suspend/reactivate/admin-invite, logged as `account.invite`) with actor, timestamp, before/after state (SR-11, BR-8). System actions use the reserved system actor id.
- Never log secrets, passwords, tokens, or full session cookies. Log authz denials and admin adjudications for review.

### Deployment & CI/CD safety
- Ship a single hardened Docker image: non-root user, minimal base, no build tools in the runtime layer, pinned base digest.
- CI: run dependency vulnerability scan and secret scan on every PR; block on known unpatched CVEs (ARCHITECTURE.md Dependency Rules). No secrets in CI logs; use the platform secret store.
- Terraform topology, secret store, and runtime hardening beyond the image: TO BE DECIDED.

### GDPR / privacy
- Data minimization: collect only email + optional profile fields (NFR-3). Email and exact location never exposed on public scoreboard.
- Support data export and account deletion/anonymization for the user's personal data; preserve audit-log integrity by anonymizing actor references rather than deleting audit rows (reconcile retention NFR-4 vs. erasure — exact policy TO BE DECIDED).
- Define lawful basis, retention schedule, and data-region. TO BE DECIDED with deployment.

## Prompt Placeholders To Resolve

Resolved from the known stack where REQUIREMENTS.md/ARCHITECTURE.md fix it; otherwise marked TO BE DECIDED.

- `{{CODE_QUALITY_PROMPT}}` → RESOLVED: low cyclomatic and cognitive complexity; separation of concerns (route → service/domain → repository per ARCHITECTURE.md). Pure, testable domain functions for state machine, score grammar, ranking.
- `{{API_SECURITY_PROMPT}}` → RESOLVED: OWASP API Security Top 10 + OWASP REST Security Cheat Sheet. Focus: BOLA/object-level authz, function-level authz, mass-assignment guards, resource/rate limits, explicit status codes.
- `{{BACKEND_FRAMEWORK_PROMPT}}` → RESOLVED (Flask): https://flask.palletsprojects.com/en/stable/web-security/ — secure cookies, CSRF, security headers, no debug in prod, escape on render, safe redirects.
- `{{FRONTEND_FRAMEWORK_PROMPT}}` → RESOLVED (React): https://relevant.software/blog/react-js-security-guide/ — avoid `dangerouslySetInnerHTML`, rely on JSX escaping, no secrets in bundle, validate before render, dependency hygiene.
- `{{AUTH_PROMPT}}` → RESOLVED: OWASP Authentication + Session Management cheat sheets, plus WebAuthn/MFA for admins. Player password hashing (Argon2id), admin passkeys, token hygiene, session lifecycle.
- `{{DEPLOYMENT_PROMPT}}` → PARTIAL: Docker image only for now (hardened, non-root, pinned). Full deployment/Terraform topology and secret store TO BE DECIDED.

## Selected Prompt Imports (by decision)

- Architecture decisions (React SPA + Flask REST + MySQL, server-as-sole-enforcer) → `{{API_SECURITY_PROMPT}}`, `{{CODE_QUALITY_PROMPT}}`.
- Backend framework (Flask) → `{{BACKEND_FRAMEWORK_PROMPT}}`.
- Frontend framework (React) → `{{FRONTEND_FRAMEWORK_PROMPT}}`.
- Auth model (player password + admin passkey/WebAuthn) → `{{AUTH_PROMPT}}`.
- Deployment model (Docker now; Terraform later) → `{{DEPLOYMENT_PROMPT}}` (PARTIAL — topology UNKNOWN).

Unresolved / UNKNOWN until inputs land: email provider, WebAuthn library, session/CSRF library, cache layer, C3 DB-enforcement technique, audit-log immutability mechanism, secret store, CI/CD platform, GDPR data-region and retention-vs-erasure policy.
