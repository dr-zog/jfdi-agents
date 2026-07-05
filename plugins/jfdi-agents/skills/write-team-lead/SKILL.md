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

**A lead is.** The one agent that can spawn teammates, create tasks, and coordinate. It is a **conductor** — it surveys state, decides the next piece of work, delegates, and waits. Its work product is the orchestrated state of the other agents, not an artefact of its own.

**A lead is not.** A worker. A consultant. A commentator on how the teammates are doing. A substitute for a teammate whose task is blocked. If the lead is authoring text, writing files, or committing code, the prompt is wrong.

## The canonical lifecycle (one stage, one persistent team)

As of Claude Code v2.1.178, `TeamCreate` and `TeamDelete` are removed. There is exactly one team per session, formed automatically on the first `Agent` spawn and cleaned up automatically at session exit. The lead does not create or delete the team; it only adds teammates.

```
Agent spawn(s)        ← spawn new teammates for the stage. First spawn of the
    │                    session implicitly forms the team (name: session-<hash>).
    │                    Persistent roles carry over; ephemeral roles are added.
    ▼
TaskCreate × N        ← one task per teammate, with owner set to the teammate's name.
    │                    Spawn first, then tasks — avoids a race where an early
    │                    teammate SendMessages a peer that doesn't exist yet.
    ▼
Poll loop             ← lead ticks TaskList every ~120s via ScheduleWakeup and
    │                    reacts to state changes. NOT waiting on SendMessage.
    ▼
Audit                 ← check the artefacts on disk match what the task promised.
    │
    ▼
Shutdown-request      ← for ephemeral teammates whose role is finished. Persistent
   (optional)           roles (product-owner, architect, repo-steward) carry on.
```

**The lead's job is strictly four things:** `Agent` spawns, `TaskCreate`/`TaskUpdate`, the poll loop, and shutdown requests for finished ephemeral teammates. If the lead wants a teammate to do something, it raises a task. It does not DM instructions.

## Tool-schema details that are easy to miss

### `Agent` tool — spawning a teammate

Required by schema: **`description`** (3–5 word label) and **`prompt`** (the task brief).

Parameters commonly passed for teammate spawns:

- `description` — **required**, short label.
- `prompt` — **required**, the self-contained task brief for the spawned teammate.
- `subagent_type` — **required for teammate spawns**. Reference the agent definition's frontmatter `name`. Passing the full body text as `prompt` instead produces an ad-hoc spawn with no role metadata (visible as `You are …` in the exit menu rather than `Role: X. Team: session (…)`).
- `team_name` — accepted but ignored (as of v2.1.178). The team is fixed as `session-<first-8-of-session-id>`. Passing something else does not create a different team.
- `name` — the teammate's name within the team (how others address it via `SendMessage`). Must be unique within the team.
- `model` — pass explicitly on every spawn. Per [anthropics/claude-code#30703](https://github.com/anthropics/claude-code/issues/30703), teammate frontmatter `model` is not reliably honoured; spawn-time `model` wins.

### `TaskCreate` — raising a task for a teammate

Key fields:

- `title` — one-line summary.
- `owner` — **the teammate's name** (the `name` you passed to `Agent`), not the `subagent_type`. This is the single most common mistake.
- `description` — self-contained task brief. The teammate reads this; it is what shapes the work. Carry all per-task runtime context here — artefact paths, consult targets, Feature IDs — so the teammate's body can stay pure domain expertise.

The lead creates tasks; **teammates transition their own tasks' status** as a harness-native behaviour. The lead should not `TaskUpdate` on a teammate's behalf, and the teammate's prompt body should not contain an explicit "TaskUpdate your task to completed" instruction — that is process injection for something the harness already does.

### Team lifecycle — no explicit create or delete

- **`TeamCreate` and `TeamDelete` no longer exist** (removed in Claude Code v2.1.178). The team is implicit: formed on the first `Agent` spawn, named `session-<first-8-of-session-id>` by the harness, cleaned up automatically at session exit. Do not write lead prompts that call these tools.
- **Session resumption is different from zombie recovery.** Under `/resume` or `/rewind`, the harness restores the task list but not in-process teammates. The lead's recovery is to read `TaskList`, reconcile with on-disk state, and re-spawn only the teammates needed for the next work (persistent roles + the current-task owner). See `${CLAUDE_PLUGIN_ROOT}/agents/team-lead.md` § "Session recovery — resumption".
- **Env flag.** `Agent`, `TaskCreate`, `TaskUpdate`, `SendMessage`, `ScheduleWakeup` (for the lead's poll loop) are all gated on `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. The lead should check the env var at session start and refuse to proceed if unset.

## Addressing discipline — silent failure if wrong

**The lead is addressed as `team-lead`, not by its role name.** The harness assigns the lead the addressable identity `team-lead@<team-name>` (where `<team-name>` is the auto-generated `session-<hash>`). `SendMessage({to: "team-lead", ...})` reaches the lead. `SendMessage({to: "TeamLead", ...})` returns `success: true`, writes to a dead-letter inbox, and emits a preview summary in the sender's idle stream — the lead sees the summary but never the message body. Silent structural bug.

A lead prompt that tells teammates to address it must use `to: "team-lead"`, with an explicit callout that the role name is the wrong value.

## Naming teammates

The lead picks names. Conventions used in this plugin:

- **Role-only** for persistent specialists: `product-owner`, `architect`, `repo-steward`. One of each per session — they carry across every stage.
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
| Closing a stage without `git status --porcelain` check | Leaves uncommitted files on the branch when a teammate's closing commit was missed | Check before advancing to the next stage; investigate dirty tree; raise a follow-up task for the owning agent. |
| `TeamCreate` / `TeamDelete` calls | Both tools removed in Claude Code v2.1.178 | The team forms implicitly on the first `Agent` spawn (harness names it `session-<hash>`) and is auto-cleaned at session exit. |
| Passing full agent body text as the `prompt` argument to `Agent` | Produces an ad-hoc spawn with no role metadata; shows raw prompt text in the exit menu | Reference the agent definition by `subagent_type`; the `prompt` argument is role-orientation only. |

## Commit discipline

Leads generally **do not author commits** — commit work belongs to whichever agent wrote the files. If the lead notices uncommitted work at a stage boundary, it raises a follow-up task assigned to the owning agent so the commit is still agent-authored under that agent's `--author=`.

The canonical rules (the `--author=` form, slug, cadence) live in the **commit-as-agent** skill. The lead body references the skill rather than restating the rules.

## System-prompt composition for the lead

The plugin's `jfdi-agent` output style strips the coding-focused base prompt for every agent in the project (main-session lead and teammates alike) while retaining Claude Code's agent-harness behaviour. The lead launches with the standard `claude --agent <name>` command — no special flags, no per-launch composition tricks.

Lead-specific consequences:

- **No coding-focused ambient prime.** No need to counter-prime, no need to override the default's "prefer editing over creating" or "do not write .md files" directives.
- **Claude Code agent-harness behaviour is retained.** `# Environment`, `# Custom Agent Instructions` boundary, `<system-reminder>` delivery — all still present.
- The lead body still carries identity, mission, state machine, spawn briefs, hard rules, the `team-lead` addressing callout, wait-semantics with human-escalation fallback, and ephemeral-teammate shutdown-request mechanics.

**Read `${CLAUDE_PLUGIN_ROOT}/docs/system-prompt-composition.md`** for the curated reference.

## Structure of a good lead body

The body shape that has held up well:

1. **Read-these-first list** — pointers to `docs/terminology.md`, `docs/roster.md`, `docs/process.md`, `docs/team-lead-playbook.md`, and any stage-specific docs.
2. **Mission** — one sentence, conductor-flavoured (*"drive the workflow autonomously", "sequence the team, never build"*).
3. **Operating modes** — if there are more than one (Checkpointed vs Autonomous JFDI).
4. **The state machine** — each stage names the teammates it adds, the ones that carry over from prior stages, its prerequisites, and its completion criterion.
5. **Per-stage spawn briefs** — one per stage the lead drives. Each brief is a template for the *task description* (the work brief the teammate reads) filled in at runtime with task-specific context (layer, phase, acceptance-item ids, artefact paths). The spawn prompt argument itself stays role-orientation only.
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
