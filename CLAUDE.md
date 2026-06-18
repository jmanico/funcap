# CLAUDE.md

Guidance for Claude Code working in this repository. Compact by design. This file holds **behavior, guardrails, and working style only** — it does not restate requirements, architecture, or security rules. Those live in the imported documents below and are the source of truth.

## Required context (import; error if missing)

These imports are mandatory. If any target file is missing, **stop and report the missing file** rather than proceeding — the repo is misconfigured.

- @REQUIREMENTS.md — what the system must do (source of truth).
- @ARCHITECTURE.md — stack, boundaries, dependency rules.
- @SECURITY.md — security rules and guardrails.

Do not duplicate or paraphrase rules from these files here. Cite them by id (FR-/SR-/BR-/NFR-/RD-) when relevant.

## Project snapshot

Funcap is a web + API application: React SPA → Flask REST API → MySQL (3NF). The repository is in **bootstrap mode** — currently only governance docs exist (REQUIREMENTS.md, ARCHITECTURE.md, SECURITY.md, REQUIREMENT_TEMPLATE.md). No application code yet. Do not infer or invent workflows, tooling, or conventions that are not written down.

## Guardrails

- REQUIREMENTS.md wins on any conflict; ARCHITECTURE.md and SECURITY.md govern their domains. Never contradict them — if a change requires it, update the doc in the same PR and flag it.
- Provisional/bootstrap content is labeled as such in the docs. Treat `TO BE DECIDED` / `UNKNOWN` markers as open — do not silently resolve them; ask or leave them marked.
- The Flask API is the sole enforcement point; never rely on the React client for authorization or validation (see SECURITY.md).
- Honor the Dependency Rules in ARCHITECTURE.md before adding any library: prefer zero new dependencies and justify any addition in the PR.
- Do not commit secrets. Do not weaken a labeled security rule without an explicit, approved change to SECURITY.md.

## GitHub issues

- **Every new GitHub issue MUST follow REQUIREMENT_TEMPLATE.md** so each issue is a structured, testable requirement (stable ID, acceptance criteria, standards alignment, test strategy). Do not open free-form issues for requirements.
- Assign the `REQ-[MODULE]-[###]` ID and fill acceptance criteria as independently testable Given/When/Then.

## Working style

- Be surgical: smallest change that satisfies the requirement; match surrounding style once code exists.
- Trace work back to a requirement id; if none exists, propose one (via the template) before implementing.
- Keep docs compact and high-signal — they are loaded into Claude Code context.
- Verify before claiming done; report failures honestly.

## Workflow — TO BE DECIDED (placeholders, do not assume)

These are not yet defined. Do not invent them; confirm with the maintainer when they become relevant.

- Branching & PR policy (branch naming, review gates, merge strategy): TO BE DECIDED.
- Test framework, runner, and coverage gate (ARCHITECTURE.md targets exist; tooling not chosen): TO BE DECIDED.
- Lint/format/type-check commands and pre-commit hooks: TO BE DECIDED.
- CI/CD pipeline (build, scan, deploy) and environments: TO BE DECIDED.
- Local dev setup / run commands (no app code yet): TO BE DECIDED.
- Commit message convention: TO BE DECIDED.
