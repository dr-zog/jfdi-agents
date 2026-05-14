---
name: write-solo-agent
description: Author or audit a solo agent prompt — a Claude Code agent designed to run as a main session paired one-to-one with a human, outside the JFDI team loop, while still respecting the team's process boundaries. **Human-gated**: this skill is invoked only after the human has explicitly approved minting a solo agent (either by direct request or by approving an Architect-led proposal that the team has discussed with the ProductOwner). The skill itself does not enforce the gate; the workflow does. Use this when the human has authorised solo work for visual polish, security review, performance investigation, exploratory research, or a focused refactor against a single subsystem. The skill bakes in the boundary discipline that lets a solo session integrate cleanly with subsequent team work — the agent knows where it fits in the ecosystem and refuses to break invariants the team established.
---

# Write a solo-agent prompt

A **solo agent** is a Claude Code main-session agent the human launches directly via `claude --agent <name>`, paired one-to-one with a human reviewer in a single session. It is *not* a teammate — there is no TeamLead, no `SendMessage` relay, no shared task list. `AskUserQuestion` is enabled because the human is right there.

This skill is a sibling of [`write-agent`](../write-agent/SKILL.md) (which handles **teammate** prompts spawned by the TeamLead) and [`write-team-lead`](../write-team-lead/SKILL.md) (the conductor pattern). Solo agents complete the trio: lead, teammate, solo.

## Before you invoke this skill — the human-gating check

**Solo agents are minted only at the human's explicit request.** The Architect does NOT unilaterally mint them, even when they look like a good fit for the work at hand. Two routes legitimately reach this skill:

1. **The human asked directly.** They said *"please write me a solo agent for X"* or invoked the skill themselves. Proceed.
2. **The Architect proposed, the team discussed, the human approved.** The Architect noticed a candidate, SendMessaged the ProductOwner to refine scope, the team agreed on a proposal, the TeamLead put the proposal to the human via `AskUserQuestion`, and the human said yes. Proceed.

Anything else — *"I'll just mint one quickly, the team will be fine with it"*, *"the work obviously needs solo treatment so I'll write the agent and present it as a fait accompli"* — is **not authorised**. Stop and route through the protocol. See `${CLAUDE_PLUGIN_ROOT}/docs/solo-agents.md` for the gating protocol in full.

This skill does not enforce the gate; the workflow and the agent bodies that invoke this skill (Architect's `proposing solo agents` section) do. If you reach this skill without one of the two authorisation routes above, the gate has been bypassed and the resulting agent is illegitimate.

## When solo mode is the right shape

Solo mode is licensed when the work has properties that don't fit team coordination:

- **The feedback loop is visual or sensory.** UI polish, layout iteration, accessibility hygiene, audio/video tuning. The human looks at the rendered output; the team-loop overhead of architect-mediated decisions is wasted motion.
- **The work is exploratory, not deliverable-shaped.** Performance investigation, security audit, GDPR research, dependency triage. The output is "we now understand X" and possibly a handover doc — not a green acceptance item.
- **The change set is dense within one subsystem.** A focused refactor inside a single layer's folder where pulling in the architect plus a teammate developer plus the verifier produces more friction than insight.
- **The cycle time is interactive.** The human is iterating in seconds, not minutes. A team-spawn round trip dominates.

Solo mode is *not* licensed when the work would naturally cross folder boundaries, change contracts other layers depend on, add new acceptance items, or is large enough that a team-coordinated build with verifier sign-off would be safer. In those cases, route through the team — that's what the team is for.

## The governing principle — two licensed paths

This is the load-bearing rule. Every solo agent body must encode it.

> **When a solo session faces work whose scope exceeds what the human session can land cleanly, it has exactly two licensed paths:**
>
> **(a) Do the work yourself, to the same standard the team would have applied.** Match the team's discipline on tests, documentation, decisions log, commit cadence. Leave the system in a state the next team session would have produced.
>
> **(b) Stop and produce a handover artefact.** Document what you discovered, what you propose, what's been done already, and what the team needs to do to integrate it. The next team session picks up the handover as input and lands the change properly.
>
> **What is never licensed is the half-state** — a partial change that breaks invariants without documenting them. A red test left red, a contract drift uncommented in `docs/decisions.md`, a cross-folder edit with no record. That state is what destroys autonomous-team confidence; the solo agent's body must refuse to leave it.

Encode this principle as prose in the agent body, not as a procedural checklist. The agent's job is to internalise it and apply judgment, not to run a flowchart.

## What to INCLUDE in a solo-agent body

1. **Identity** — one sentence, naming the work shape and the fact that this is a solo, human-paired agent. *"You are X, a human-paired Y agent for project Z. You operate outside the team-of-agents flow."*
2. **Read-these-first list, with the team's contracts at the top.** The vision (especially `vision/acceptance.md`), `docs/architecture.md`, `docs/decisions.md`, then the project-specific files the agent will actually edit. Stress that these define the contract the solo session must not break.
3. **Mission** — domain framing. What the agent iterates on with the human, what success looks like.
4. **What you may edit** — strict scope. File paths and folders. If the agent may touch test files, name the conditions (see § "The mechanism-vs-contract distinction").
5. **What you must NOT edit** — explicit list with reasons. Adjacent layers, infrastructure files (conftest, migrations config), architecture docs, dependency manifests, contract surfaces (API URLs, schemas, auth mechanics).
6. **The mechanism-vs-contract distinction** if the agent touches tests — the load-bearing pattern that lets the agent rename selectors without regressing behaviour.
7. **Coordination with the human** — the cycle: human points at problem, agent proposes scoped change, human reviews, agent commits. `AskUserQuestion` is enabled; use freely.
8. **Branch and commit discipline** — feature branch, small commits, human-not-agent identity on commits, human (not agent) does the merge.
9. **Test-running discipline** — inherit the team's standards (chunking, time limits, never-skip rules). Red tests are a stop signal.
10. **Hard boundaries — moments to stop and surface** — the explicit list of things that route to the human or to a handover artefact.
11. **Closing the session** — what to write at the end. Either a "completed work" record or a "handover" record (see § "Closing the session").
12. **What you are not** — identity statements naming each team role and what makes the solo agent different from each.

## What to EXCLUDE from a solo-agent body

| Exclude | Why | Where it belongs |
|---|---|---|
| "When the TeamLead spawns you..." | Topology that doesn't apply — there is no TeamLead in a solo session. | Nowhere. |
| "SendMessage the architect for cross-folder questions." | The team isn't running. The human is the routing authority. | Replace with: "Surface to the human via `AskUserQuestion`; if the question is architectural, propose a handover artefact for the team." |
| "TaskUpdate when complete." | Harness-native and task-list-relative, not relevant outside a team. | Nowhere. |
| Procedural step-by-step "first do X, then Y, then Z." | Solo work is iterative with the human. Encoding a sequence makes the agent rigid. | Replace with the principle (two licensed paths + boundaries) and trust the human-agent cycle. |
| "You are part of stage 4 — Refine." | Workflow-stage knowledge that doesn't apply outside the team session. | Nowhere. |
| Generic safety prose like "be careful." | Not actionable. | Replace with concrete boundaries — *"do not modify any `fetch(...)` URL or method; those are backend contracts"*. |

## The mechanism-vs-contract distinction (load-bearing if the agent touches tests)

Many solo agents legitimately need to edit tests in the same commit as the work they're polishing — a CSS selector rename forces the test that observes that selector to update. This is fine *if and only if* the test continues to assert the same observable behaviour, only the mechanism by which it observes has changed.

The distinction the agent body must encode:

- **Mechanism is implementation.** Selectors, IDs, class names, internal helper signatures, file paths. Renamable as long as the test still observes the same user-facing outcome.
- **Contract is invariant.** HTTP routes, request/response shapes, schema field names, auth headers, public API symbols, anything `vision/acceptance.md` describes as observable behaviour. Never changed in a solo session.

The litmus test the agent should apply before any "rename a thing the tests reference" change:

> *"Does the test still observe the same user-facing behaviour, just through a different name?"* If yes, proceed and update the test in lock-step. If the answer drifts toward *"…but the user no longer sees X"* or *"…and we removed the assertion that Y"*, the change has crossed from polish into regression. Stop.

**Never make a test pass by deleting an assertion.** If deleting the assertion is the only way to ship, the change is wrong.

State this rule plainly in any solo agent body that may rename test-referenced symbols. It's the single most common way a solo session damages the team's invariants.

## Frontmatter expectations

```yaml
---
name: <kebab-case>
description: Single-session, human-paired <role> agent for <project context>. Operates outside the team-of-agents flow. <Bounded to which folders or domains>. <Explicit non-scope statement, e.g. "Never touches behaviour-bearing code or contracts".>
model: sonnet         # opus only if the work warrants it; solo work is usually sonnet
color: <unique>
---
```

**`AskUserQuestion` is NOT in `disallowed-tools`.** Solo agents have it enabled — that's the whole point. Do not paste the teammate frontmatter's `disallowed-tools: AskUserQuestion` line into a solo agent.

**No `hooks`, `mcpServers`, `permissionMode`** — same constraints as plugin-shipped agents and minted teammates.

## Branch and commit discipline

Solo agents commit as the human, not as a synthetic identity. This is deliberate:

- **Author is the human's git config.** The commit IS human-in-loop work; attributing it to a synthetic `frontend-polish@jfdi-agents.invalid` would lie about who reviewed it.
- **The human (not the agent) does the merge to `main`.** `git merge --no-ff feature/<slug>` mirrors the RepoSteward pattern and produces a clean audit trail boundary between solo work and team work.
- **No `commit-as-agent` skill** — that's the teammate flow. Solo agents commit directly with `git commit -m "..."` and inherit the human's identity.

The agent body should explicitly say: *"Don't author as `<role>@anything.invalid` — this is human-in-loop work."*

Branches: `feature/<slug>` matching the team convention (e.g. `feature/polish-1`, `feature/security-audit-q2`).

## Hard boundaries — moments to stop and surface

Every solo agent body needs an explicit "stop and ask the human" list, tailored to the agent's domain. The shape:

- **Adding a dependency** of any kind.
- **Editing outside the agent's permitted folders.**
- **Renaming a symbol that the agent has not first verified is unreferenced** by tests in adjacent layers, by route handlers, by schemas.
- **Modifying any contract surface** — API call shapes, schemas, auth mechanics, anything cross-layer.
- **Anything that "feels like a small refactor while I'm here."** Solo agents are not refactor agents; explicit anti-scope-creep clause.
- **Discovering that an acceptance item or ADR is genuinely ambiguous or wrong.** Surface; do not silently re-interpret.

Use `AskUserQuestion` for these. If the harness blocks `AskUserQuestion` for any reason, the agent should fall back to writing the question as plain output — never silently proceed past a boundary.

## Closing the session — completed work or handover

A solo session has two valid endings, both producing a written record under `docs/demos/<date>-solo-<slug>.md`:

### Ending A — completed work

The work is done, all relevant test chunks green, the human is satisfied. The record names:
- Date, branch, HEAD, what the session set out to do.
- Acceptance items unchanged (table) — confirming no regressions.
- Concrete changes by file.
- Tests touched, with each row noting whether the change was a mechanism rename (acceptable) or contract change (which would have triggered a stop, so this column should never see a contract entry).
- Test-suite summary — chunked counts, pass/fail.
- Findings — usually "no findings"; if the session surfaced a concern outside polish scope, note it as a positive find for a future team-driven pass.

### Ending B — handover

The work is in progress and exceeds what the solo session should land. The record names:
- Date, branch, HEAD, what the session set out to do, **what's still outstanding**.
- What the agent did already (committed work, scoped to safe changes).
- What the agent discovered that needs broader treatment (decisions, contract changes, acceptance-item clarifications).
- A concrete proposal for the team to act on — which agent role would own the next step (Architect for an ADR, ProductOwner for an acceptance-list edit, a layer dev for implementation).
- Acceptance items unchanged — same table as Ending A, confirming nothing regressed despite the work being incomplete.

Either way, the agent **does not merge** — the human reviews the record, decides whether to merge the branch (Ending A) or leave it for a future team session to integrate (Ending B).

## What the agent is NOT

Every solo-agent body should include an explicit identity-by-contrast section. It heads off the most common drift — a solo agent slowly behaving like a team role:

- **Not a TeamLead.** No team, no spawned teammates, no relay protocol.
- **Not an Architect.** Does not write to `docs/architecture.md` or `docs/decisions.md` directly. Architectural questions surface to the human or land in a handover for a future team session.
- **Not a Verifier.** Demo reports written by solo agents are first-person narratives of the polish/audit/investigation, not independent verifier audits.
- **Not the team-spawned `<role>-dev`.** That agent ships behaviour against acceptance items, lives within the team flow, commits as a synthetic identity, and has `AskUserQuestion` disabled. The solo agent is its complement.

## Tools and skills

- **All standard Claude Code tools** are available (Read, Edit, Write, Bash, Grep, Glob).
- **`AskUserQuestion` is enabled.** Use freely.
- **Domain-specific Anthropic skills** that fit the work (frontend-design, security-review, etc.) — the agent body should name the relevant ones explicitly so the human knows what to enable.
- **Git operations are direct** — no `commit-as-agent` skill; no synthetic author.

## Structure of a good solo-agent body

The shape mirrors `write-agent`'s teammate template, with five differences:

1. Frontmatter — `AskUserQuestion` is permitted (no `disallowed-tools` line).
2. The mission section explicitly names "outside the team-of-agents flow" and "human-paired".
3. The two-licensed-paths principle is the central organising idea, stated as prose.
4. The "what you must NOT edit" list is heavier than for teammates — the absence of a TeamLead routing back means the agent is the last line of defence against scope creep.
5. The closing protocol allows for two endings (completed work or handover) instead of the teammate's `TaskUpdate(status: "completed")` signal.

## How to audit an existing solo-agent body

Read top to bottom, flagging:

- **Topology leaks.** Any "TeamLead", "SendMessage", "TaskUpdate", "the architect will" — solo agents don't see those. Replace with "the human" or "a handover artefact".
- **Stage-awareness language.** "in the Build stage", "during Refine pass 3" — wrong frame; solo sessions are stage-orthogonal.
- **Missing two-licensed-paths principle.** If the body has hard boundaries but no positive prescription for what to do *when work exceeds scope*, the agent will either freeze or bulldoze. Add the two-paths rule.
- **Frontmatter with `disallowed-tools: AskUserQuestion`.** That's a teammate frontmatter line that's been mistakenly copied; remove.
- **Synthetic author guidance.** If the body says to commit as `<role>@<something>.invalid`, that's the teammate flow — strip it.
- **No "what you are not" section.** Add one — without it, the agent drifts toward team-role behaviour.
- **No closing-record format.** Add it — both endings (completed work and handover) need explicit shape.

A well-written solo-agent body is roughly the same length as a teammate's — not shorter. The "what you are not" and "two licensed paths" sections add weight that compensates for the missing team-coordination prose.

## Worked example — the polish agent

The canonical first instance of this pattern is a UI-polish agent for a multi-tenant admin app: a human launches it after the team has built the system, iterates visually, and commits scoped layout / CSS / accessibility changes. Hits all the patterns this skill describes:

- Bounded to one folder (`frontend/`) plus matched test files (under the mechanism-vs-contract rule).
- Refuses to touch contracts (HTTP calls, schemas, auth).
- Uses `AskUserQuestion` freely.
- Commits as the human; the human merges.
- Has an explicit "what you are not" naming each team role.
- Closes with a polish-pass demo record.

But polish is just one instance. The same skill mints solo agents for security audit, performance investigation, GDPR research, focused refactors — anywhere 1:1 human attention is the right shape. The discipline is identical; the domain prose changes.

## References (canonical)

- `${CLAUDE_PLUGIN_ROOT}/docs/roster.md` — the team's roles, for the "what you are not" section.
- `${CLAUDE_PLUGIN_ROOT}/docs/terminology.md` — vocabulary the solo agent must speak when interfacing with team artefacts (acceptance list, layers, folder map, demo).
- `${CLAUDE_PLUGIN_ROOT}/docs/solo-agents.md` — the canonical reference for solo agents: when licensed, the gating protocol, integration discipline, the two-licensed-paths principle. This skill authors against that doc.
- `${CLAUDE_PLUGIN_ROOT}/skills/write-agent/SKILL.md` — sibling skill for teammate agents.
- `${CLAUDE_PLUGIN_ROOT}/skills/write-team-lead/SKILL.md` — sibling skill for the conductor.
- `${CLAUDE_PLUGIN_ROOT}/docs/system-prompt-composition.md` — output-style behaviour applies to solo agents the same as teammates.
