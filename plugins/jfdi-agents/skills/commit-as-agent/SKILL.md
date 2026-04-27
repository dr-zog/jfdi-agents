---
name: commit-as-agent
description: How an agent in the jfdi-agents team commits its work. Single source of truth for the `--author=` rule, the stage-end commit-cadence rule, the Build-phase per-layer and Refine-pass per-item exception, and per-role staging patterns. Agent bodies reference this skill rather than restating the rules.
---

# Commit as an agent

Every agent that writes files commits its own work. This skill is the canonical reference for how. Agent bodies reference it rather than restating the rules.

## The canonical form

```bash
git commit --author="<TeammateName> <slug@jfdi-agents.invalid>" -m "<type>(<scope>): <subject>

<body>"
```

- **`--author=`** sets the git **author** field to the agent; the **committer** field stays as the human's configured identity (so GPG signing, push permissions, and platform identity all work unchanged).
- **`<TeammateName>`** is the name the TeamLead assigned at spawn time (e.g., `product-owner`, `architect`, `backend-dev-skeleton`, `frontend-dev-refine-3`, `verifier-skeleton-data`). Not the subagent_type. **All-lowercase kebab-case** per the team-naming convention in `${CLAUDE_PLUGIN_ROOT}/docs/team-lead-playbook.md` § 1.2.
- **`<slug>`** is the teammate name verbatim (it is already lowercase and already has no colons or spaces): `backend-dev-skeleton` → `backend-dev-skeleton`, `architect` → `architect`.
- The `.invalid` TLD is reserved by RFC 2606 for non-deliverable addresses. Using it means the author email looks sensible but cannot accidentally be a real address.

## Why this matters

- Without `--author=`, every agent commit shows up in `git log` as if the human wrote it. After session compaction, the TeamLead reads the log, sees the human's name next to work the agent did, and concludes the human is actively driving — wrong behaviour follows.
- The git blame / history record becomes dishonest. You cannot tell which agent produced which code.
- Post-incident analysis (who wrote the broken commit — was it the backend-dev or the frontend-dev?) becomes impossible.

## Examples per role

- **product-owner** at end of Stage 1 (Intake):
  ```bash
  git commit --author="product-owner <product-owner@jfdi-agents.invalid>" -m "docs: initial Vision

  - vision/overview.md
  - vision/goals.md
  - vision/constraints.md
  - vision/personas.md
  - vision/glossary.md"
  ```

- **architect** at end of Stage 2 (Architecture & team design), including the minted developer agents:
  ```bash
  git commit --author="architect <architect@jfdi-agents.invalid>" -m "feat: architecture, acceptance list, and minted developer agents

  - docs/architecture.md
  - docs/decisions.md
  - vision/acceptance.md (co-authored with product-owner — see trailer)
  - .claude/agents/data-dev.md
  - .claude/agents/backend-dev.md
  - .claude/agents/frontend-dev.md

  Co-authored-by: product-owner <product-owner@jfdi-agents.invalid>"
  ```

- **backend-dev-skeleton** during Build (per layer milestone, typically multiple commits):
  ```bash
  git commit --author="backend-dev-skeleton <backend-dev-skeleton@jfdi-agents.invalid>" -m "feat(backend): scaffold Fastify app with /health route

  - backend/src/app.ts
  - backend/src/server.ts
  - backend/package.json"
  ```

- **frontend-dev-refine-3** during Refine (per acceptance item made real):
  ```bash
  git commit --author="frontend-dev-refine-3 <frontend-dev-refine-3@jfdi-agents.invalid>" -m "feat(frontend): implement acceptance #7 — rename task

  - frontend/src/components/TaskItem.tsx
  - frontend/src/hooks/useTaskRename.ts"
  ```

- **verifier-skeleton-complete** writing the demo:
  ```bash
  git commit --author="verifier-skeleton-complete <verifier-skeleton-complete@jfdi-agents.invalid>" -m "docs: skeleton-complete demo — Ready-to-advance: Yes

  - docs/demos/2026-04-25-skeleton-complete.md"
  ```

- **repo-steward** merging a feature branch (local-only flow):
  ```bash
  # git merge --no-ff itself produces the merge commit; the --author flag is passed to the merge command
  git merge --no-ff feature/skeleton-backend \
    -m "merge feature/skeleton-backend" \
    --author="repo-steward <repo-steward@jfdi-agents.invalid>"
  ```

## Commit cadence — per stage

The workflow has discrete stages. Cadence varies:

| Stage | Cadence | Who commits |
|---|---|---|
| Intake | 1× at end | `product-owner` |
| Architecture & team design | 1× at end, bundling all Stage 2 artefacts | `architect` (with `product-owner` as Co-authored-by: for acceptance.md) |
| Build — per layer | N× per layer (one per meaningful milestone, minimum one) | `<layer>-dev-skeleton` |
| Build — per layer demo | 1× | `verifier-skeleton-<layer>` |
| Build — complete demo | 1× | `verifier-skeleton-complete` |
| Refine pass | N× per pass (one per acceptance item made real, developers interleave) | the various `<layer>-dev-refine-<N>` working in parallel |
| Refine pass demo | 1× per pass | `verifier-refine-<N>` |
| Merge commits | At stage close | `repo-steward` |

See `${CLAUDE_PLUGIN_ROOT}/docs/process.md` § "Commit cadence" for the full table.

## What each role stages

- **product-owner** — `vision/*.md` files; in Stage 2, also contributes to `vision/acceptance.md` as a co-author of Architect's commit.
- **architect** — `docs/architecture.md`, `docs/decisions.md`, `.claude/agents/*-dev.md`; in Stage 2, the bundled commit co-authored with product-owner also includes `vision/acceptance.md`.
- **`<layer>-dev-<phase>`** — files inside their owned folder only (`<folder>/...`). `git status` after their commits should show changes scoped to that folder.
- **verifier-<phase>** — `docs/demos/<date>-<slug>.md`.
- **repo-steward** — nothing of its own. Merge commits only (via `git merge --no-ff`).

## When the commit fails

- **Pre-commit hook failure:** the commit did not happen. Fix the underlying issue, re-stage, create a **new** commit. Never `--amend` a failed commit.
- **`--author=` looks wrong in `git log`:** you probably forgot to pass the flag. No fix other than a new commit with the right author; git log history is immutable.
- **No changes to commit:** that means a previous stage already committed the files, or you are running a consultation that produces no file output. Do not create an empty commit.

## Hard rules

- **Never skip hooks** (`--no-verify`, `--no-gpg-sign`, etc.) unless the human has explicitly asked for it.
- **Never force-push to `main`.** Ever.
- **Never update the git config.**
- **Stage specific files by name** rather than `git add -A` or `git add .` — those can accidentally include sensitive files (`.env`, credentials) or large binaries.
- **One logical change per commit.** If you find yourself wanting to bundle unrelated edits, split them.
- **Developers: stage only files inside your owned folder.** A commit that touches another folder is the coherence failure mode Verifier exists to catch — fix it before pushing rather than after.

## Reference

- `${CLAUDE_PLUGIN_ROOT}/docs/process.md` § "Git discipline" — branching rules, which agents create branches, merge flow.
- `${CLAUDE_PLUGIN_ROOT}/docs/process.md` § "Commit cadence" — the full stage-by-stage flow.
