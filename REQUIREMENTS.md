# Funcap: Community Tennis Tournament Application

Status: Draft v0.2
Owner: Carlo
Purpose: Authoritative, testable, machine-readable requirements for the Funcap Tennis web application. This document is the contract that ARCHITECTURE.md, SECURITY.md, CLAUDE.md, and the test suite derive from. Every requirement has a stable ID for traceability to tests.

---

## 1. Overview

Funcap is a web application for running a recurring series of community tennis tournaments. Each tournament runs for a fixed three-month window. Players register for free, enroll in a tournament, find opponents through a matchmaking flow, play, and self-report results. Results are confirmed by the opponent. A public scoreboard shows standings. Disputed results are resolved by an admin.

The system is trust-based: players report their own scores and their own skill level. Trust is backed by three controls: mutual confirmation of every result, an immutable audit trail, and an admin backstop for disputes.

## 2. Glossary

- Player: a registered user who can enroll in tournaments and play matches.
- Admin: a privileged user who manages tournaments and resolves disputes. An admin may also be a player. When an admin is a participant in a match, they cannot administer that match (resolve its dispute, set its final result, or void it); a different admin must act. See BR-9.
- Tournament: a time-boxed competition (three months) with its own standings.
- Enrollment: a player's participation record in a specific tournament.
- Match: a single official contest between two enrolled players within one tournament.
- Result: the set scores recorded for a match (for example 6-2, 3-6, 6-2).
- Confirmed match: a match whose result has been approved by the opponent, auto-confirmed by timeout, or resolved by an admin, and that counts toward standings. A result confirmed by explicit opponent approval, or resolved by an admin, is locked and final to players. A result auto-confirmed by timeout remains escalatable to admin review until the tournament completes (RD-5). Only admins can change locked results.
- Pair: an unordered set of two distinct players, `{playerX, playerY}`.
- Standings: ranked ordering of players within one tournament, computed from confirmed and resolved matches.
- Scoreboard: the public, read-only presentation of standings.

## 3. Actors and Roles

| Role | Capabilities |
|------|--------------|
| Anonymous visitor | View the public scoreboard. Register as a player. Authenticate. Nothing else. |
| Player | All of the above, plus: manage own profile, enroll in tournaments, propose and respond to matches, submit and approve results, raise disputes on matches they participated in. |
| Admin | All player capabilities, plus: create and manage tournaments, resolve disputes and set final results, suspend or reactivate accounts, void results. Subject to separation of duties (BR-9). |

Roles are inclusive in privilege (admin is a superset of player). Authorization is deny-by-default: an action is forbidden unless a rule explicitly grants it.

## 4. Data Model

This section defines logical entities and the constraints that must hold. Field-level types and storage choices belong in ARCHITECTURE.md; the constraints below are normative.

### 4.1 User

- `id` (immutable, system-generated)
- `email` (unique, used for login and verification)
- `password_hash` (never the plaintext; nullable)
- `email_verified_at` (nullable)
- `role` (`player` or `admin`)
- `status` (`active`, `suspended`)
- `created_at`

Authentication note: players authenticate with email and password, so `password_hash` is populated for player accounts. Admins authenticate with a passkey and have no password by default (FR-6, SR-18), so `password_hash` is null for passkey-only admin accounts. The credential storage model (password records, WebAuthn credential records) is an ARCHITECTURE.md concern; the normative point is that `password_hash` is nullable and an account may have a password credential, a passkey credential, or both.

### 4.2 Profile

One-to-one with User.

- `display_name` (shown publicly on the scoreboard)
- `skill_self_rating` (self-reported, NTRP scale 1.0 to 7.0 in 0.5 steps, optional)
- `location` (optional, free text or region code)
- `bio` (optional, free text)

`skill_self_rating` is a profile attribute used for display and matchmaking hints only. It does not affect standings.

### 4.3 Tournament

- `id`
- `name`
- `starts_on`, `ends_on` (the three-month window; `ends_on` is derived as `starts_on` + 3 months unless overridden by admin)
- `status` (`upcoming`, `active`, `completed`)
- `created_by` (admin user id)
- Constraint C4 (non-overlapping windows): no two tournaments whose status is in {`upcoming`, `active`} may have overlapping `[starts_on, ends_on]` windows. This enforces RD-1 (sequential, at most one active at a time) at creation and on any admin date override.

### 4.4 Enrollment

Links one User to one Tournament.

- `id`
- `user_id`
- `tournament_id`
- `enrolled_at`
- Constraint: unique on (`user_id`, `tournament_id`). A player enrolls in a given tournament at most once.

### 4.5 Match

- `id`
- `tournament_id`
- `proposer_id` (the player who initiated)
- `opponent_id` (the player who was challenged)
- `status` (see Section 6 state machine)
- `pair_low` (derived: `min(proposer_id, opponent_id)`)
- `pair_high` (derived: `max(proposer_id, opponent_id)`)
- `proposed_at`, `accepted_at`, `played_on` (nullable per state)
- Constraint C1 (no self-match): `proposer_id != opponent_id`.
- Constraint C2 (both enrolled): both players have an Enrollment for `tournament_id` at the time the match becomes `accepted`.
- Constraint C3 (one official match per pair per tournament): a partial unique index on (`tournament_id`, `pair_low`, `pair_high`) restricted to matches whose status is in {`accepted`, `score_submitted`, `confirmed`, `disputed`, `resolved`}. This is the database-level enforcement of business rule BR-3. Matches in `proposed`, `declined`, or `cancelled` do not occupy the pairing.

`pair_low` and `pair_high` exist so the uniqueness constraint is order-independent: a match A-vs-B is the same pairing as B-vs-A.

### 4.6 Result

One-to-one with a Match (a match has at most one active result).

- `id`
- `match_id`
- `sets` (ordered list of set scores, each a pair of non-negative integers, for example `[[6,2],[3,6],[6,2]]`)
- `winner_id` (derived from `sets`)
- `submitted_by` (player id)
- `submitted_at`
- `approved_by` (player id, nullable; null when the result was auto-confirmed)
- `approved_at` (nullable; null when the result was auto-confirmed)
- `confirmation_source` (`opponent_approval` or `auto`, set when the match becomes `confirmed`; null before confirmation). When `auto`, `approved_by` and `approved_at` are null. This field gates the escalation transition in Section 6 and enforces RD-5 and BR-10.

### 4.7 Dispute

- `id`
- `match_id`
- `raised_by` (player id, must be a participant)
- `reason` (free text, required)
- `status` (`open`, `resolved`)
- `resolved_by` (admin id, nullable; must not be a participant in the match, BR-9)
- `resolution` (admin's final set scores and notes, nullable)
- `created_at`, `resolved_at`

### 4.8 Audit Log

Append-only. No update or delete by any role.

- `id`
- `actor_id` (the user id of the actor; for system actions such as expiry and auto-confirmation, a reserved system actor id)
- `action` (enumerated: `match.propose`, `match.accept`, `match.decline`, `match.cancel`, `match.expire`, `result.submit`, `result.approve`, `result.reject`, `result.auto_confirm`, `result.void`, `dispute.raise`, `dispute.resolve`, `account.suspend`, `account.reactivate`, etc.)
- `subject_type`, `subject_id` (the entity acted on)
- `payload` (before and after state where applicable)
- `occurred_at`

`result.auto_confirm` is distinct from `result.approve` so an auto-confirmation is always distinguishable from an explicit approval (FR-19). `match.expire` records proposal auto-cancellation (FR-15). Escalation of an auto-confirmed result reuses `dispute.raise`, since it opens a Dispute record.

## 5. Functional Requirements

### 5.1 Accounts and Authentication

- FR-1: A visitor can register as a player with email and password. Verify: a player account is created in `active` status with `email_verified_at` null.
- FR-2: The system sends an email verification link on registration. Verify: clicking a valid, unexpired link sets `email_verified_at`.
- FR-3: A player must have a verified email before enrolling in a tournament or proposing a match. Profile editing is allowed before verification. Verify: enrollment and match proposal are rejected when `email_verified_at` is null.
- FR-4: A player can authenticate with email and password and can log out. Verify: logout invalidates the session.
- FR-5: A player can reset a forgotten password via an emailed, single-use, time-limited token. Verify: the token cannot be reused and expires.
- FR-6: An admin account is created only by invitation from an existing admin, and authenticates with a passkey (WebAuthn). There is no self-service admin registration and no admin password by default. The first admin is provisioned out-of-band during deployment (seed configuration), not through the application. Verify: a registration attempt that would create an `admin` role without a valid admin invitation is rejected; a non-admin cannot issue an admin invitation; admin authentication uses a registered passkey. Ref: SR-18.

### 5.2 Profile

- FR-7: A player can view and edit their own profile fields (`display_name`, `skill_self_rating`, `location`, `bio`). Verify: a player cannot edit another player's profile (object-level authorization).
- FR-8: `display_name` is required and must be non-trivial for the scoreboard. Verify: empty or whitespace-only display names are rejected.

### 5.3 Tournaments and Enrollment

- FR-9: An admin can create a tournament with a name and start date. The end date defaults to start plus three months. Verify: a non-admin cannot create a tournament; creation is rejected if the proposed window overlaps an existing `upcoming` or `active` tournament (C4).
- FR-10: A tournament transitions `upcoming` to `active` on or after `starts_on`, and `active` to `completed` on or after `ends_on`. Because tournaments are sequential and non-overlapping (RD-1, C4), at most one tournament is `active` at a time. Transition mechanism (scheduled job or on-access evaluation) is an ARCHITECTURE.md concern; the observable behavior is normative.
- FR-11: A verified player can self-enroll in an `upcoming` or `active` tournament. Verify: enrollment in a `completed` tournament is rejected.
- FR-12: Only players enrolled in a tournament can participate in matches in that tournament. Verify: a non-enrolled player cannot be a proposer or opponent in that tournament.

### 5.4 Match Lifecycle (Matchmaking)

- FR-13: An enrolled player (proposer) can propose a match to another enrolled player (opponent) in the same active tournament. Verify: the proposal is created in `proposed` status and constraints C1 and C2 are enforced.
- FR-14: The opponent can accept or decline a proposal. Verify: accept moves the match to `accepted` and triggers the C3 uniqueness check; decline moves it to `declined`.
- FR-15: A proposal that is neither accepted nor declined within a configurable window (default 14 days) is automatically expired to `cancelled` and recorded with `match.expire`. Verify: an expired proposal does not occupy the pairing (RD-5).
- FR-16: Either participant can cancel a match while it is in `proposed` or `accepted` status (before any result is submitted). Verify: cancellation frees the pairing and is recorded in the audit log.

### 5.5 Result Entry and Approval

- FR-17: For an `accepted` match, either participant can submit a result consisting of the set scores. Verify: the match moves to `score_submitted`, `submitted_by` is recorded, and the score passes the validation grammar in Section 7.
- FR-18: The participant who did not submit the result can approve or reject it. Verify: the submitter cannot approve their own submission (BR-5). Approval moves the match to `confirmed` and sets `confirmation_source = opponent_approval`. Rejection moves the match to `disputed` and opens a Dispute record with `raised_by` set to the rejecting participant and a required reason.
- FR-19: A submitted result that is neither approved nor rejected within a configurable window (default 7 days) is auto-confirmed: the match moves to `confirmed` with `confirmation_source = auto`, recorded with the distinct action `result.auto_confirm`. An auto-confirmed result remains escalatable to admin review until the tournament completes (RD-5, BR-10, Section 6). Verify: auto-confirmation is distinguishable in the audit log from explicit approval, and an auto-confirmed match can be escalated by either participant before tournament completion.
- FR-20: `winner_id` is derived from the set scores by the rules in Section 7, never entered directly. Verify: the derived winner matches the set majority.
- FR-21: A result that is `confirmed` by explicit opponent approval, or `resolved`, is immutable to players. A result that is `confirmed` by auto-confirmation (`confirmation_source = auto`) is immutable to players except via the escalation transition in Section 6, which is available until the tournament completes. Only an admin can void or correct a confirmed or resolved result (FR-26). Verify: players receive a forbidden response on direct edit attempts to confirmed or resolved results.

### 5.6 Disputes and Admin Resolution

- FR-22: A participant can raise a dispute. There are two entry paths. First, rejecting a submitted result (FR-18) moves the match to `disputed` and opens a Dispute. Second, either participant can escalate an auto-confirmed result to admin review, which moves the match from `confirmed` to `disputed` and opens a Dispute, at any time before the tournament completes. In both paths the Dispute record is created with `status = open` and a required reason. Verify: only a participant can raise or escalate a dispute on their own match; escalation is rejected for results confirmed by explicit approval, and rejected after the tournament has completed.
- FR-23: An admin can view all open disputes with the full match context and audit history. Verify: the dispute queue is admin-only.
- FR-24: An admin who is not a participant in the match can resolve a dispute by setting the final set scores (or voiding the match) and recording a resolution note. This moves the match to `resolved`. Verify: resolution writes to the audit log with `dispute.resolve`; an admin who is a participant in the match is forbidden from resolving it (BR-9, SR-17).
- FR-25: A `resolved` match counts toward standings using the admin's final scores. Verify: standings recompute to reflect the resolution.
- FR-26: An admin who is not a participant in the match can void a confirmed result (for example on discovery of collusion) and can suspend or reactivate accounts. Verify: a voided result no longer counts toward standings, the action is audited, and an admin who is a participant in the match is forbidden from voiding it (BR-9, SR-17).

### 5.7 Standings and Scoreboard

- FR-27: The system computes standings for each tournament from confirmed and resolved matches only, using the ranking specification in Section 8. Verify: matches in any non-final state, and voided results, do not affect standings.
- FR-28: A public, read-only scoreboard displays the current standings of the single active tournament to anyone, including anonymous visitors. When no tournament is active (a gap between sequential tournaments), the scoreboard displays the most recently completed tournament's frozen final standings with a clear indicator that it is the most recent completed tournament, and offers a selector for historical tournaments. Verify: no authentication is required to view; no private data (email, exact location) is exposed.
- FR-29: The scoreboard shows, per player: rank, display name, wins, losses, and the tiebreak metrics from Section 8. Verify: ordering matches the ranking specification exactly.
- FR-30: Completed tournaments remain viewable as historical standings. Verify: a completed tournament's final standings are frozen and stay accessible.

## 6. Match State Machine

States: `proposed`, `accepted`, `declined`, `score_submitted`, `confirmed`, `disputed`, `resolved`, `cancelled`.

Terminal states: `declined`, `cancelled`, `resolved`, and `confirmed` when `confirmation_source = opponent_approval`. A `confirmed` match with `confirmation_source = auto` is terminal only after the tournament completes; before then it can be escalated to `disputed`.

States that occupy a pairing (count against BR-3): `accepted`, `score_submitted`, `confirmed`, `disputed`, `resolved`.

| From | Event | Trigger (role) | To | Guard |
|------|-------|----------------|----|-------|
| (none) | propose | proposer | proposed | C1, C2, both enrolled, tournament active |
| proposed | accept | opponent | accepted | C3 uniqueness passes, tournament still active |
| proposed | decline | opponent | declined | - |
| proposed | cancel | either participant | cancelled | - |
| proposed | expire | system (14d) | cancelled | audited as `match.expire` |
| accepted | submit result | either participant | score_submitted | score passes Section 7 grammar |
| accepted | cancel | either participant | cancelled | no result yet |
| score_submitted | approve | the non-submitter | confirmed | BR-5; sets `confirmation_source = opponent_approval` |
| score_submitted | reject | the non-submitter | disputed | BR-5; opens Dispute (raised_by = rejecter, reason required) |
| score_submitted | auto-confirm | system (7d) | confirmed | sets `confirmation_source = auto`; audited as `result.auto_confirm` (RD-5) |
| confirmed | escalate (request admin review) | either participant | disputed | `confirmation_source = auto` and tournament not completed; opens Dispute (BR-10) |
| disputed | resolve | admin | resolved | acting admin is not a participant (BR-9); admin sets final result or voids |
| confirmed | void | admin | confirmed (result flagged voided, match retained, excluded from standings) | acting admin is not a participant (BR-9) |
| resolved | void | admin | resolved (result flagged voided, match retained, excluded from standings) | acting admin is not a participant (BR-9) |

Void is an action that flags the active result as voided so it no longer counts toward standings; the match is retained for audit. Any transition not listed is forbidden. Implementations must reject illegal transitions with a clear error and must not silently no-op.

## 7. Result Validation Grammar

Scores are validated as a business rule, not merely a format check. Invalid scores are rejected at submission (FR-17) and at admin resolution (FR-24).

Rules for v0.2 (RD-3): best-of-three sets, standard tennis scoring with a tiebreak at 6-6, and a full third set (a 10-point match tiebreak is not used). The grammar below applies uniformly to all three sets.

A result is an ordered list of two or three set scores. Each set score is an ordered pair `[a, b]` of non-negative integers, where `a` is the proposer's games and `b` is the opponent's games (orientation is fixed by the match, not the submitter).

A single set is valid if and only if one of the following holds (symmetrically for either player as the set winner):

- `6-x` where `0 <= x <= 4` (a clean set won at 6).
- `7-5` (won by two after 5-5).
- `7-6` (won via tiebreak after 6-6).

No other set score is valid. In particular `6-5`, `8-6`, `6-6`, and any score with a margin or magnitude outside the above are rejected.

Match-level validity:

- The match winner is the first player to win two sets.
- A two-set result is valid only if the same player won both sets (a 2-0 result). The list must then have exactly two sets.
- A three-set result is valid only if each player won exactly one of the first two sets and one player won the third (a 2-1 result). The list must then have exactly three sets.
- A result that would imply a match continued after a player already reached two set wins is invalid (for example, three sets where the same player won the first two).

`winner_id` is derived as the player with two set wins. Submitters never set the winner directly (FR-20).

## 8. Ranking Specification

Standings are computed per tournament from confirmed and resolved matches only. The ordering must be deterministic and reproducible from the match data, so that any developer can recompute and verify it.

Per player, compute over their confirmed and resolved matches in the tournament:

- `W` = matches won
- `L` = matches lost
- `played` = `W + L`
- `set_diff` = sets won minus sets lost
- `game_diff` = games won minus games lost
- `win_pct` = `W / played` (defined as 0 when `played` is 0)

Ordering, applied as a cascade (each criterion breaks ties left by the previous one):

1. `W` descending (more match wins ranks higher).
2. `win_pct` descending (efficiency tiebreak for unequal numbers of matches played).
3. `set_diff` descending.
4. `game_diff` descending.
5. Head-to-head: for a tie between exactly two players who played each other, the winner of that match ranks higher.
6. Final deterministic tiebreak: earlier `created_at` of the user account ranks higher. This guarantees a total, reproducible order with no ambiguity.

This is the resolution of RD-2 (match wins primary). The rationale and the override path are in Section 13.

### 8.1 Worked Example

Confirmed matches:

- Alice beat Bob 6-2, 6-3
- Alice beat Carol 6-4, 3-6, 6-2
- Bob beat Carol 7-5, 6-4
- Dave beat Bob 6-1, 6-0

Computed:

| Player | W | L | win_pct | set_diff | game_diff |
|--------|---|---|---------|----------|-----------|
| Alice | 2 | 0 | 1.00 | +3 | +10 |
| Dave | 1 | 0 | 1.00 | +2 | +11 |
| Bob | 1 | 2 | 0.33 | -2 | -14 |
| Carol | 0 | 2 | 0.00 | -3 | -7 |

Resulting order: 1 Alice, 2 Dave, 3 Bob, 4 Carol.

Dave (1 win from 1 match) outranks Bob (1 win from 3 matches) because, tied on `W`, `win_pct` favors Dave. This is the direct consequence of ranking on match wins first, then efficiency. It is intentional under RD-2.

## 9. Security Requirements

Security requirements are first-class and testable. They align with OWASP ASVS 5.0 and the referenced OWASP Cheat Sheets. The full threat model and control implementation detail belong in SECURITY.md; the requirements below are the normative must-haves.

### 9.1 Authentication and Session

- SR-1: Passwords are stored using a memory-hard, salted algorithm (Argon2id preferred, bcrypt acceptable). Plaintext or fast unsalted hashes are prohibited. Applies to player password credentials; admins are passkey-only by default (SR-18). Ref: Password Storage Cheat Sheet.
- SR-2: Authentication endpoints enforce rate limiting and lockout-resistant throttling to resist credential stuffing and brute force. Ref: Authentication Cheat Sheet.
- SR-3: Sessions use server-issued tokens delivered in cookies marked `HttpOnly`, `Secure`, and `SameSite`. Sessions have an idle timeout and an absolute timeout, and are invalidated on logout and on password change. Ref: Session Management Cheat Sheet.
- SR-4: Password reset and email verification tokens are single-use, time-limited, and unpredictable. Ref: Forgot Password Cheat Sheet.

### 9.2 Authorization

- SR-5: Access control is deny-by-default and enforced server-side on every state-changing and data-returning endpoint. Client-side checks are never the enforcement point. Ref: Authorization Cheat Sheet.
- SR-6: Object-level authorization is enforced on all match, result, and dispute operations. A player may submit, approve, reject, or dispute only matches in which they are a participant. This is the primary defense against broken object level authorization (BOLA/IDOR), which is the highest-risk class for this app because rankings derive from these objects. Ref: OWASP API Security Top 10, Authorization Cheat Sheet.
- SR-7: Function-level authorization restricts tournament management, dispute resolution, account suspension, and result voiding to admins. Ref: Authorization Cheat Sheet.

### 9.3 Business Logic Integrity

- SR-8: The submitter of a result cannot approve it (enforces BR-5). Verify with a test that attempts self-approval and expects a forbidden response.
- SR-9: The one-official-match-per-pair-per-tournament rule (BR-3) is enforced at the database level via constraint C3, not only in application code, to prevent races and bypass.
- SR-10: Illegal state-machine transitions (Section 6) are rejected server-side.
- SR-11: All score-affecting actions (submit, approve, reject, auto-confirm, dispute, resolve, void) are written to the append-only audit log (Section 4.8) with actor, timestamp, and before/after state. The audit log is immutable to all roles.
- SR-17: Separation of duties is enforced on admin adjudication. The admin performing a dispute resolution (FR-24), a result void (FR-26), or a final-score setting on a match must not be a participant in that match: the acting admin's id must differ from both `proposer_id` and `opponent_id`. Enforced server-side. Ref: Authorization Cheat Sheet, ASVS access control (separation of duties).

### 9.4 Input Validation and Output Encoding

- SR-12: All input is validated server-side against an allowlist or grammar (set scores per Section 7, NTRP rating per its scale, enumerations for status fields). Ref: Input Validation Cheat Sheet.
- SR-13: User-controlled fields displayed on the public scoreboard or profiles (`display_name`, `bio`, `location`) are contextually output-encoded to prevent stored cross-site scripting. A Content Security Policy is deployed as defense in depth. Ref: Cross Site Scripting Prevention Cheat Sheet, Content Security Policy Cheat Sheet.
- SR-14: State-changing requests are protected against cross-site request forgery (anti-CSRF tokens or equivalent for cookie-based sessions). Ref: CSRF Prevention Cheat Sheet.

### 9.5 Transport and Configuration

- SR-15: All traffic is served over TLS; HTTP is redirected to HTTPS and HSTS is set. Ref: Transport Layer Security Cheat Sheet.
- SR-16: Error responses do not leak stack traces, internal identifiers, or whether an email is registered (uniform responses on auth and reset flows). Ref: Error Handling Cheat Sheet.

### 9.6 Admin Account Provisioning

- SR-18: Admin accounts are invite-only and authenticate with phishing-resistant WebAuthn passkeys. There is no self-service admin registration and no admin password by default. An admin invitation can be issued only by an existing admin. The first admin is provisioned out-of-band during deployment (seed configuration) rather than through the application. Ref: Authentication Cheat Sheet, Multifactor Authentication Cheat Sheet.

## 10. Business Rules (Invariants)

These invariants must hold at all times. Each maps to enforcement points above.

- BR-1: A player cannot play themselves. Enforced by C1.
- BR-2: Both players in a match must be enrolled in that tournament at acceptance. Enforced by C2.
- BR-3: At most one official match exists per unordered pair per tournament. Enforced by C3 and SR-9.
- BR-4: Only confirmed and resolved matches affect standings, and a voided result does not count. Enforced by FR-27.
- BR-5: The player who submits a result cannot be the player who approves it. Enforced by SR-8.
- BR-6: Standings ordering is total and deterministic. Enforced by Section 8 step 6.
- BR-7: A result confirmed by explicit opponent approval, or resolved, is immutable to players. A result auto-confirmed by timeout is immutable to players except via escalation to admin review (BR-10). Only admins can void or correct any confirmed or resolved result. Enforced by FR-21 and FR-26.
- BR-8: The audit log is append-only. Enforced by SR-11.
- BR-9: An admin who is a participant in a match cannot adjudicate that match (resolve its dispute, set its final result, or void it). A different admin must act. Enforced by SR-17. Operational dependency in NFR-5.
- BR-10: An auto-confirmed result (confirmation by timeout) is not final to players until the tournament completes. Either participant may escalate it to admin review, which moves the match to `disputed`. Explicitly approved results are final to players. Enforced by FR-19, FR-22, and the Section 6 escalation transition.

## 11. Non-Functional Requirements

- NFR-1: The public scoreboard is the highest-read surface and must remain available and performant under read load; standings may be cached or materialized provided they reflect confirmed results within a bounded staleness (target under 60 seconds).
- NFR-2: All times are stored in UTC; tournament windows and timeouts are evaluated in UTC.
- NFR-3: Personal data is limited to what the application needs (email, optional profile fields). Email is never shown publicly.
- NFR-4: The audit log is retained for the lifetime of the tournament series and is exportable for admin review.
- NFR-5: At least two admin accounts should exist so that separation of duties (BR-9) can always be satisfied. If only one admin exists and that admin is a participant in a disputed match, the dispute remains `open` until a second, non-participant admin is available to resolve it. Deployments are expected to provision a second admin before the first admin enrolls as a player.

## 12. Out of Scope (Non-Goals for v0.2)

- Payments, fees, or prizes.
- In-app messaging or chat between players.
- Court or facility booking and scheduling.
- Computed skill ratings such as Elo or Glicko (skill is self-reported only; see RD-2).
- Doubles or team play (singles only in v0.2).
- Native mobile applications (responsive web only).
- Cross-tournament aggregate or all-time leaderboards beyond viewing historical per-tournament standings (see RD-4).
- Concurrent or overlapping tournaments (see RD-1).
- A 10-point match tiebreak third-set format (see RD-3).
- A player-initiated dispute channel for results confirmed by explicit approval; such results are final to players, and suspected fraud on them is handled by admin void (FR-26).

## 13. Resolved Decisions

Each decision below was an open question in v0.1. The decision is now normative and propagated into the sections above. The rationale and override path are recorded so the product owner can confirm or reverse a choice deliberately. RD-2 is the one decision the owner should consciously sign off, because it shapes player incentives.

### RD-1: Tournament overlap (was OD-1)

Decision: Tournaments are strictly sequential and non-overlapping. At most one tournament is `active` at a time. Enforced by constraint C4 at creation and on any date override.

Rationale: This matches a single community running one competition at a time, keeps the public scoreboard unambiguous (one active tournament, FR-28), and removes the need for a tournament selector on the active view. Historical tournaments remain selectable.

Override path: To allow concurrent tournaments, remove C4, allow multiple `active` tournaments, and add a tournament selector to the scoreboard. FR-28 would then show a chosen active tournament rather than the single one.

### RD-2: Ranking incentive (was OD-2)

Decision: Match wins primary, with win percentage as the first tiebreak, per the cascade in Section 8.

Rationale: Wins-primary maximizes the incentive to play matches, which is the lifeblood of a recurring community series. It is fully deterministic and trivially verifiable from match data. BR-3 (one match per pair per tournament) prevents farming a single weak opponent, so climbing requires playing different opponents and winning. The known consequence, that an undefeated low-volume player can be outranked by a more active player with more total wins, is intentional for community play: showing up and winning repeatedly is rewarded over playing two matches and stopping. Early in a tournament a 1-0 player can transiently top the board; this resolves as active players accumulate wins.

Override path (the alternative the owner may prefer): rank by win percentage primary with a minimum-matches qualification gate. Concretely, introduce a configurable `min_ranked_matches` (suggested 3 to 5). Players with `played < min_ranked_matches` are shown in a separate provisional section and are not assigned a ranked position. Among qualified players, reorder the Section 8 cascade to `win_pct` descending first, then `W`, then `set_diff`, then `game_diff`, then head-to-head, then account `created_at`. This crowns the most efficient qualified player rather than the most active winner. It is a one-section change to Section 8 plus the qualification gate and a scoreboard provisional section. A points system (for example 3 per win) is equivalent to wins-primary when all wins weigh equally and is not recommended unless weighted bonuses are introduced.

### RD-3: Third-set format (was OD-3)

Decision: Full third set, with a tiebreak at 6-6. A 10-point match tiebreak is not used. Section 7 grammar is unchanged and applies uniformly to all three sets.

Rationale: A single uniform grammar across all sets is simpler to implement, validate, and audit, and is fully deterministic. Players self-schedule, so the time savings of a match tiebreak are not needed.

Override path: To use a 10-point match tiebreak for the third set, add a distinct third-set grammar to Section 7: the third set is recorded as a tiebreak score `[a, b]` where the winner reaches at least 10 and wins by at least 2 (for example `10-7`, `12-10`), and the match-level rules treat the third-set tiebreak as the deciding set. Set and game differentials would need a defined treatment for the tiebreak set (for example, count it as one set won and the tiebreak points as games, or exclude it from `game_diff`); this must be decided to keep Section 8 deterministic.

### RD-4: Scope of "global" scoreboard (was OD-4)

Decision: "Global" means publicly visible standings for the single active tournament, plus frozen historical per-tournament standings selectable by anyone. No all-time cross-tournament aggregate leaderboard in v0.2. Consistent with Section 12.

Rationale: Per-tournament standings are the unit of competition. An all-time aggregate is a distinct feature with its own ranking semantics and is not required to ship v0.2.

Override path: Add an all-time aggregate leaderboard as a future feature with its own defined aggregation rule (for example total wins across the series, or a qualification-gated all-time win percentage). This is additive and does not change v0.2 behavior.

### RD-5: No-response defaults (was OD-5)

Decision: Unanswered proposals auto-cancel after a configurable window (default 14 days, FR-15), and unapproved results auto-confirm after a configurable window (default 7 days, FR-19). Both are recorded with distinct audited actions (`match.expire` and `result.auto_confirm`). A result confirmed by auto-confirmation (`confirmation_source = auto`) remains escalatable to admin review by either participant until the tournament completes; escalation moves the match from `confirmed` to `disputed`. A result confirmed by explicit opponent approval is final to players.

Rationale: Auto-confirmation keeps standings moving in a trust model where an inattentive opponent would otherwise freeze a result indefinitely. Distinguishing auto-confirmation from explicit approval, and keeping auto-confirmed results escalatable until the tournament ends, prevents a silent timeout from becoming an irreversible loss for an honest but inattentive opponent, while still letting the result count immediately. This resolves the tension between FR-19 (auto-confirm) and FR-21 (immutability to players).

Override path: To never auto-confirm, remove the `score_submitted` to `confirmed` auto-confirm transition and require an explicit approval. Results would then remain in `score_submitted` indefinitely until approved, rejected, or escalated to admin review. If this path is chosen, add an explicit "request admin review" transition from `score_submitted` so a stalled result is not stuck.
