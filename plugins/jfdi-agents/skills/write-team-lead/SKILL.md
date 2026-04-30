---
name: write-team-lead
description: Author or audit a team-lead agent prompt — the kind that uses Claude Code's agent-teams feature to spawn teammates, create tasks, wait, and tear down. Captures the canonical lifecycle, the tool-schema details that are easy to miss, the addressing discipline that silently fails when wrong, and the lead anti-patterns we have paid for across the plugin's release history. Use this skill any time you are creating, editing, refactoring, or auditing a lead agent prompt in `plugins/jfdi-agents/agents/` (today the canonical lead is `team-lead.md`; one-off test leads also count), or when an incident has surfaced a new lead-side failure mode that needs locking down. Pair with `write-agent` when the other side of the team is also being authored, or with `write-solo-agent` when minting an agent that operates outside the team loop entirely. Living document — update it whenever a new lesson is learned the hard way.
---

# Write a team-lead agent prompt

A **team lead** is an agent that manages a Claude Code [agent team](https://code.claude.com/docs/en/agent-teams): it creates the team, spawns specialists as named teammates, raises tasks for them, waits for completion, and tears the team down at stage boundaries. In this plugin the canonical team lead is `TeamLead`, but any agent whose job is to conduct other agents follows the same shape.

This skill is the checklist / reference / audit tool for writing that kind of prompt. It is a **living document** — update it whenever we learn a new lesson the hard way.

## When to run this

- You are authoring a new team-lead agent from scratch.
- You are refactoring `team-lead.md` (or a one-off test lead).
- You are auditing an existing lead prompt against the canonical shape.
- An incident has surfaced a new lead-side failure mode and we need to lock the lesson down.

## Mental model — what a lead is and is not

**A lead is.** The one agent that can create a team, add members, create tasks, and delete the team. It is a **conductor** — it surveys state, decides the next piece of work, delegates, and waits. Its work product is the orchestrated state of the other agents, not an artefact of its own.

**A lead is not.** A worker. A consultant. A commentator on how the teammates are doing. A substitute for a teammate whose task is blocked. If the lead is authoring text, writing files, or committing code, the prompt is wrong.

## The canonical lifecycle (one stage)

```
TeamCreate
    │
    ▼
Agent spawns          ← spawn first, then tasks. Race condition otherwise.
    │
    ▼
TaskCreate × N        ← one task per teammate, with owner set to the teammate's name.
    │
    ▼
Wait                  ← tasks reach `completed`. Lead reads state from disk, does NOT poll teammates.
    │
    ▼
Audit                 ← check the artefacts on disk match what the task promised.
    │
    ▼
TeamDelete            ← clean up. Pre-teardown + post-teardown `git status --porcelain` if stage wrote files.
```

**The lead's job is strictly four things:** `TeamCreate`, `Agent` spawns, `TaskCreate`/`TaskUpdate`, wait. If the lead wants a teammate to do something, it raises a task. It does not DM instructions.

## Tool-schema details that are easy to miss

### `Agent` tool — spawning a teammate

Required by schema: **`description`** (3–5 word label) and **`prompt`** (the task brief).

Parameters commonly passed for teammate spawns:

- `description` — **required**, short label.
- `prompt` — **required**, the self-contained task brief for the spawned teammate.
- `subagent_type` — the agent definition's frontmatter `name`.
- `team_name` — the team the spawned teammate joins. **If omitted, `Agent` spawns a plain subagent**, not a teammate — a common silent failure.
- `name` — the teammate's name within the team (how others address it via `SendMessage`). Must be unique within the team.
- `model` — pass explicitly on every spawn. Per [anthropics/claude-code#30703](https://github.com/anthropics/claude-code/issues/30703), teammate frontmatter `model` is not reliably honoured; spawn-time `model` wins.

### `TaskCreate` — raising a task for a teammate

Key fields:

- `title` — one-line summary.
- `owner` — **the teammate's name** (the `name` you passed to `Agent`), not the `subagent_type`. This is the single most common mistake.
- `description` — self-contained task brief. The teammate reads this; it is what shapes the work. Carry all per-task runtime context here — artefact paths, consult targets, Feature IDs — so the teammate's body can stay pure domain expertise.

The lead creates tasks; **teammates transition their own tasks' status** as a harness-native behaviour. The lead should not `TaskUpdate` on a teammate's behalf, and the teammate's prompt body should not contain an explicit "TaskUpdate your task to completed" instruction — that is process injection for something the harness already does.

### `TeamCreate` / `TeamDelete`

- **Zombie teams.** `TeamDelete` returns `success: true` even on a zombie. Recovery is manual `rm -rf "$STATE_DIR/teams/<name>/"` followed by `TeamCreate` replay, where `$STATE_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"`. Aliveness check: `find "$STATE_DIR/teams/<name>/" -mmin -10` distinguishes live from zombie. Hard-coding `~/.claude` silently misses the per-project-isolation case (see `${CLAUDE_PLUGIN_ROOT}/agents/team-lead.md` § "Session recovery — aliveness check" for the verified-name + isolation-warning protocol).
- **Env flag.** `TeamCreate` / `TeamDelete` / `SendMessage` are hard-gated on `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. The lead should check the env var at session start and refuse to proceed if unset.

## Addressing discipline — silent failure if wrong

**The lead is addressed as `team-lead`, not by its role name.** `TeamCreate` returns `lead_agent_id: "team-lead@<team-name>"`; `SendMessage({to: "team-lead", ...})` reaches the lead. `SendMessage({to: "TeamLead", ...})` returns `success: true`, writes to a dead-letter inbox, and emits a preview summary in the sender's idle stream — the lead sees the summary but never the message body. Silent structural bug.

A lead prompt that tells teammates to address it must use `to: "team-lead"`, with an explicit callout that the role name is the wrong value.

## Naming teammates

The lead picks names. Conventions used in this plugin:

- **Role-only** for stage-wide specialists: `product-owner`, `architect`, `repo-steward`. One of each per stage.
- **Phase suffix** for per-unit work: `backend-dev-skeleton`, `frontend-dev-refine-3`, `verifier-skeleton-data`, `verifier-refine-1`.
- **All-lowercase kebab-case** — the harness lowercases one side of its on-disk state but not the other; mixed-case team names break teammates' `TaskList` silently. See `${CLAUDE_PLUGIN_ROOT}/docs/team-lead-playbook.md` § 1.2.
- **Unique within the team and across the session.** Name collisions break `SendMessage` and can cause context contamination.
- **Only the lead spawns.** Teammates cannot add other teammates (documented harness constraint).

## Lead anti-patterns — do not re-introduce these

| Anti-pattern | Why it is wrong | Instead |
|---|---|---|
| Routine "status check" `SendMessage`s | Invites the teammate to respond, inflates the lead's context, duplicates state already in the task register | Read `TaskList` / `TaskGet`. Read artefacts on disk. |
| Ad-hoc `SendMessage` with instructions ("please do X") | Bypasses the task register; the teammate correctly ignores it as non-canonical | Raise a new task via `TaskCreate`. |
| "Just do this small thing myself" | Moves work from teammates into the lead's context; creates write pressure on an agent whose job is to conduct | Spawn the owning developer (or Architect for cross-folder decisions) — never do it yourself. |
| Name the lead by its role in `SendMessage` | Silent dead-letter; lead sees preview not body | `to: "team-lead"` with a callout. |
| `TaskUpdate` on a teammate's task | Confuses who owns status | Teammates own their own status. |
| Teardown without `git status --porcelain` check | Leaves uncommitted files on the branch when a teammate's closing commit was missed | Check before teardown; investigate dirty tree; Developer team for post-teardown cleanup. |

## Commit discipline

Leads generally **do not author commits** — commit work belongs to whichever agent wrote the files. If the lead needs commits it cannot attribute (e.g., post-teardown leftovers from closing turns), it spins up a narrow follow-up team so the commits are still agent-authored under that agent's `--author=`.

The canonical rules (the `--author=` form, slug, cadence) live in the **commit-as-agent** skill. The lead body references the skill rather than restating the rules.

## System-prompt composition for the lead

The plugin's `jfdi-agent` output style strips the coding-focused base prompt for every agent in the project (main-session lead and teammates alike) while retaining Claude Code's agent-harness behaviour. The lead launches with the standard `claude --agent <name>` command — no special flags, no per-launch composition tricks.

Lead-specific consequences:

- **No coding-focused ambient prime.** No need to counter-prime, no need to override the default's "prefer editing over creating" or "do not write .md files" directives.
- **Claude Code agent-harness behaviour is retained.** `# Environment`, `# Custom Agent Instructions` boundary, `<system-reminder>` delivery — all still present.
- The lead body still carries identity, mission, state machine, spawn briefs, hard rules, the `team-lead` addressing callout, wait-semantics with human-escalation fallback, and teardown mechanics.

**Read `${CLAUDE_PLUGIN_ROOT}/docs/system-prompt-composition.md`** for the curated reference.

## Structure of a good lead body

The body shape that has held up well:

1. **Read-these-first list** — pointers to `docs/terminology.md`, `docs/roster.md`, `docs/process.md`, `docs/team-lead-playbook.md`, and any stage-specific docs.
2. **Mission** — one sentence, conductor-flavoured (*"drive the workflow autonomously", "sequence the team, never build"*).
3. **Operating modes** — if there are more than one (Checkpointed vs Autonomous JFDI).
4. **The state machine** — each stage names its team, its teammates, its prerequisites, its completion criterion.
5. **Per-stage spawn briefs** — one per stage the lead drives. Each brief is a template the lead fills in at runtime with task-specific context (layer, phase, acceptance-item ids, artefact paths).
6. **Wait-loop / signal handling** — how the lead detects completion (`TaskList`, disk reads, teammate turns), how long-waiting is handled (read state, inspect git, escalate to human via `AskUserQuestion` — no teammate DMs), what it does on incoming teammate `SendMessage`.
7. **Hard-rules** — the conduct-only bits that are temptation-resistant.
8. **Session recovery** — post-compaction aliveness check, zombie recovery.
9. **Teardown** — pre-teardown + post-teardown checks, the narrow follow-up-team fallback for leftovers.

The TeamLead file `agents/team-lead.md` is the reference implementation. Study it before writing a new lead.

## How to audit an existing lead body

Read top to bottom, flagging:

- Any `SendMessage` reference that does not use `to: "team-lead"` for the lead or a specific teammate name for a peer.
- Any "the next agent in line" text hardcoded outside the state-machine section.
- Any "just do this" shortcut instruction that would pull the lead into authoring work.
- Any `TaskUpdate` the lead performs on behalf of a teammate.
- Any spawn-brief that lacks `description` in its `Agent` call spec.
- Any spawn-brief that does not pair with a matching `TaskCreate`.
- Any "SendMessage the teammate to check status" idle-checking text.

## References (canonical)

- `${CLAUDE_PLUGIN_ROOT}/docs/team-lead-playbook.md` — the full team-management reference.
- `${CLAUDE_PLUGIN_ROOT}/docs/process.md` § "The agent-teams tool surface" — the canonical concept → tool mapping.
- `${CLAUDE_PLUGIN_ROOT}/docs/process.md` § "Commit authorship" — the `--author=` rule.
- `plugins/jfdi-agents/agents/team-lead.md` — reference implementation of a team lead.
- `${CLAUDE_PLUGIN_ROOT}/docs/system-prompt-composition.md` — system-prompt composition findings and the output-style resolution.
