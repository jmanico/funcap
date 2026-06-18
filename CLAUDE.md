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

## Workflow — resolved conventions

These are now decided (see ARCHITECTURE.md → Selected technology decisions for rationale/versions). Follow them; do not silently deviate.

- Branching & PR policy: trunk-based with short-lived branches off `main`; every PR needs 1 approving review + green CI; **squash-merge**.
- Commit messages: **Conventional Commits** (`feat:`, `fix:`, `docs:`, `chore:`, …).
- Test framework, runner, coverage gate: backend **pytest** (+`pytest-cov`); frontend **Vitest** + React Testing Library; **≥80% branch coverage** enforced in CI.
- Lint/format/type-check + pre-commit: backend **Ruff** + **Black**; frontend **ESLint** + **Prettier** + **TypeScript** (strict); all wired via the **`pre-commit`** framework.
- CI/CD pipeline: **GitHub Actions** — tests + **pip-audit**/**npm audit** (dep CVEs) + **gitleaks** (secrets) on every PR; block on known unpatched CVEs.
- Local dev setup / run commands: established by the INFRA scaffolding issues (backend app factory, React skeleton, Docker compose for MySQL+Redis); update this line once those land.

Still TO BE DECIDED (do not invent; confirm with the maintainer): concrete email/SMTP provider, key-management/secret-store backend, Terraform topology/hosting/data-region, and GDPR retention-vs-erasure policy.
