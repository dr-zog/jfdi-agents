# TeamLead playbook

> Operational mechanics for the TeamLead role. The roster (`roster.md`) says what the TeamLead's scope is. The process doc (`process.md`) says what the overall workflow is. This document is the TeamLead's how-to: naming conventions, checkpoint cadence, stall detection, the status block format shown to the human, the exact sequence of tool calls at each stage boundary.

## 1. Conventions

### 1.1 Team name

A run of the plugin creates a Claude Code agent team. Team names are short, lowercase, kebab-case, and unique across the user's Claude Code state. The TeamLead picks a fresh team surname at session start, e.g. `jfdi-apollo`, `jfdi-mercury`, `jfdi-puffin`. Pick something memorable — it appears in log entries and in author slugs.

### 1.2 Teammate naming

All teammate names are lowercase kebab-case. Role name first, suffix last:

| Role | Teammate name pattern | Example |
|---|---|---|
| ProductOwner | `product-owner` | `product-owner` |
| Architect | `architect` | `architect` |
| Developer (minted) | `<layer>-dev-<phase>-<suffix>` | `backend-dev-skeleton`, `frontend-dev-refine-3` |
| Verifier | `verifier-<phase>-<suffix>` | `verifier-skeleton-data`, `verifier-refine-1` |
| RepoSteward | `repo-steward` | `repo-steward` (one per team) |

Naming rules:
- Lowercase kebab-case. No underscores, no camelCase.
- The phase suffix makes it obvious which run a teammate belongs to (Build-phase vs Refine-pass-N).
- A role may be spawned more than once during the project lifetime — each spawn gets a fresh suffix.

### 1.3 Commit-author slugs

Whatever the teammate name is, that's also the author slug: `<teammate-name>@jfdi-agents.invalid`. So `backend-dev-skeleton`'s commits have author `backend-dev-skeleton <backend-dev-skeleton@jfdi-agents.invalid>`. See `process.md` § "Commit authorship".

### 1.4 Branch naming

| Stage | Branch name |
|---|---|
| Vision intake | `feature/vision` |
| Architecture & team design | `feature/architecture` |
| Build — per layer | `feature/skeleton-<layer>` (e.g. `feature/skeleton-data`) |
| Refine — per pass | `feature/refine-<N>` (e.g. `feature/refine-1`) |

One branch per stage. RepoSteward creates at stage start, merges at stage close.

## 2. The TeamLead's two channels — Tasks for state, SendMessage for nudges

This split is the single most important operational rule for the TeamLead. It is also covered in `${CLAUDE_PLUGIN_ROOT}/docs/process.md` § "The two channels"; this section restates it from the lead's perspective.

**Tasks are the work-and-state channel.** All work you assign goes into `TaskCreate` with `owner`, `blockedBy`, `status: pending`, and a self-contained description. All teammate progress comes back to you via `TaskUpdate` — `in_progress` when an agent picks up, `completed` when it finishes, new `blockedBy` entries when it hits a dependency. You read this state via the periodic-poll loop (§ "The periodic-poll loop"). The Task channel is the source of truth.

**SendMessage is the nudge-and-out-of-band-Q&A channel.** You use it for:

- **Stall-check nudges** — when the periodic-poll loop's 5-minute threshold fires, SendMessage the owner of the stale `in_progress` task asking what is blocking them. One sentence. No rich brief.
- **Relaying questions** — a teammate SendMessages you with a question for the human; you `AskUserQuestion` and SendMessage the answer back.
- **Operational clarifications** — *"backend-dev: I noticed your task description says X but the architecture doc says Y, can you clarify?"*

**You do NOT use SendMessage to:**

- Tell a teammate to start work. (The Task is the assignment. If you've assigned an unblocked task to someone, they will pick it up.)
- Tell a teammate a task has been unblocked. (Clear the `blockedBy` entry via `TaskUpdate` on the downstream task; the owner will pick it up.)
- Deliver a work brief. (Put the brief in the Task description.)
- Announce that you are starting a stage, closing a stage, or any other lifecycle event. (Status blocks go to the human, not to teammates.)

### Spawn-prompt template

When you spawn a teammate via `Agent`, the prompt you pass is **orientation only**. Role + team name + a single grounding sentence. **Never** a work brief.

```
Role: <role-name> (canonical role: ProductOwner / Architect / Developer / Verifier / RepoSteward).
Team: <team-name>.
You will be assigned tasks via TaskCreate. Read your owned tasks, do the work,
TaskUpdate as you progress. Refer to your role definition for the rest.
```

That is it. No checklist, no file paths, no commit format, no completion template. The teammate's standing role definition covers the discipline; the task description covers the specifics.

### Wake-signal guidance

You rarely need to send any SendMessage to wake a teammate. Teammates pick up assigned-unblocked tasks from `TaskList` polling naturally — that is the harness's design. If you find yourself wanting to send "task #N is now unblocked, please proceed", **don't**. Clear the `blockedBy` entry via `TaskUpdate`; that is the wake-up signal.

The one legitimate wake-style SendMessage is the stall-check nudge: *"team-lead → backend-dev: task #7 has been in_progress for 5 minutes with no updates. What's blocking you?"* One sentence. No work brief.

## 3. The periodic-poll loop

Authoritative version lives in `${CLAUDE_PLUGIN_ROOT}/agents/team-lead.md` § "The periodic-poll loop". Summary for cross-reference:

- **Mechanism: `ScheduleWakeup`** (the harness's `/loop`-dynamic interface). Each tick the TeamLead does its `TaskList` + reactions, then before yielding the turn calls `ScheduleWakeup(delaySeconds: 60)` so the harness re-enters the lead for the next tick. **Not `Bash(sleep)`** — that holds one turn open across an entire stage, balloons context, and is un-interruptible.
- Every ~60 seconds, `TaskList` the team's task state.
- React to state changes — clear `blockedBy` entries when blockers complete, spawn follow-up specialists when their cue task completes, close the stage when all tasks are `completed`.
- Five consecutive ticks (~5 minutes) with no state change AND a task is `in_progress` ⇒ stall threshold fires ⇒ SendMessage the owner asking what is blocking them. One specific teammate, one sentence.
- The loop terminates when the stage is complete (no `ScheduleWakeup` call on the final tick — the turn ends naturally and the lead moves on to stage close / human checkpoint / next-stage setup).
- The existing `${CLAUDE_PLUGIN_ROOT}/scripts/stall-detector.sh` is a complementary external observer, not the primary pulse.

## 4. Start-of-session playbook

When the TeamLead main session launches, execute these steps in order.

1. **Verify environment.**
   - `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is set. If the bootstrap skill has been run, it'll be in `.claude/settings.json`. Warn the human if missing.
   - **Sandbox is enabled** in `.claude/settings.json` (`sandbox.enabled: true`). If absent, warn the human that filesystem isolation is off. Bootstrap-generated settings.json includes the sandbox by default — its absence usually means the project pre-dates bootstrap or settings.json has been edited.
   - **`CLAUDE_CONFIG_DIR` is optional**. The bootstrap-generated settings.json gives sandbox + verified-name protection without needing it. Don't warn or block on it being unset; just note its state in your status block for transparency. (If the human has set it, the zombie-recovery routine will use the scoped state tree.)
2. **Survey the project state.** Read (non-exhaustively): `vision/*` (does the Vision exist?), `docs/architecture.md` (is architecture done?), `.claude/agents/*-dev.md` (are developers minted?), `docs/demos/` (what's the last demo status?). The survey tells you which stage to enter.
3. **Decide the stage.**

   | State observed | Stage to enter |
   |---|---|
   | `vision/` empty or missing files | Stage 1 — Intake |
   | `vision/` populated but `docs/architecture.md` missing | Stage 2 — Architecture |
   | Architecture done, `.claude/agents/*-dev.md` missing | Stage 2 continuing — mint developers |
   | All minted, `docs/demos/*skeleton*` missing | Stage 3 — Build |
   | Skeleton demo exists, Ready-to-advance: Yes | Stage 4 — Refine |
   | Acceptance list fully green | Stop — project complete |

4. **Ask the human to confirm.** Via `AskUserQuestion`, confirm (a) the stage you've diagnosed is correct, (b) they want to proceed in Checkpointed or Autonomous JFDI mode. On confirmation, call `TeamCreate`.
5. **Spawn the core teammates for the stage.** See § 5 below.

## 5. Per-stage spawn playbook

### 3.1 Stage 1 — Intake

Team: `jfdi-<surname>-intake`.

Spawn:
- `product-owner` (the interactive interview)
- `repo-steward`

First-task descriptions (paste these into `TaskCreate.description` — spawn prompts stay role-orientation-only per § 2):

- **RepoSteward** — owner: `repo-steward`, blockedBy: none. Description: *"Open branch `feature/vision` from `main`. Mark this task `completed` once the branch is checked out."*
- **ProductOwner** — owner: `product-owner`, blockedBy: `[repo-steward's task above]`. Description: *"Run the Vision intake interview. Ask one question at a time via SendMessage to team-lead; team-lead will relay to the human via AskUserQuestion. Produce the five files in `vision/` described in the roster. Do not write `vision/acceptance.md` yet — that happens in Stage 2. Mark this task `completed` once the five files are committed."*

At stage close, the poll loop notices both tasks complete; TeamLead opens a final close-branch task for RepoSteward and proceeds.

### 3.2 Stage 2 — Architecture & team design

Team: `jfdi-<surname>-architecture`.

Spawn:
- `architect`
- `product-owner` (still around — co-authors the acceptance list)
- `repo-steward`

First-task descriptions:

- **RepoSteward** — owner: `repo-steward`, blockedBy: none. Description: *"Open branch `feature/architecture` from `main`. Mark this task `completed` once the branch is checked out."*
- **Architect** — owner: `architect`, blockedBy: `[repo-steward's task]`. Description: *"Read `vision/`. Author `docs/architecture.md` with the four required sections (Layers, Folder map, Developer roster, Technology stack). Seed `docs/decisions.md` with initial pinning calls. Mint one `.claude/agents/<layer>-dev.md` per layer using the `write-agent` skill. Coordinate with product-owner via SendMessage to co-author `vision/acceptance.md`. Mark this task `completed` once all four artefacts are committed."*
- **ProductOwner** — owner: `product-owner`, blockedBy: `[repo-steward's task]`. Description: *"Co-author `vision/acceptance.md` with architect — architect will SendMessage you to open the co-authoring conversation. Relay any human-scope questions to team-lead. Mark this task `completed` once `vision/acceptance.md` is committed."*

Before closing the stage: TeamLead runs a sanity check:
- `.claude/agents/*-dev.md` parseable as YAML frontmatter + body
- Each file names one folder and one language stack
- `docs/architecture.md`'s Developer roster lists the same files
- `vision/acceptance.md` has at least three items; every item is end-user observable

On any failure, route back to Architect as a `FIX:` task.

### 3.3 Stage 3 — Build (walking skeleton)

**One branch per layer.** Each layer is its own stage-team within Stage 3.

For each layer (Architect picks the order — typically data first):

Team: `jfdi-<surname>-skeleton-<layer>`.

Spawn:
- `architect` (kept from previous team's memory via kickoff brief)
- `<layer>-dev-skeleton` (the minted developer for this layer)
- `repo-steward`

Verifier (`verifier-skeleton-<layer>`) is spawned later — when the poll loop notices the developer's task transition to `completed` (and architect's approval task is also `completed`), the lead spawns Verifier and opens its task.

First-task descriptions:

- **RepoSteward** — owner: `repo-steward`, blockedBy: none. Description: *"Open branch `feature/skeleton-<layer>` from `main`. Mark this task `completed` once checked out."*
- **`<layer>-dev-skeleton`** — owner: `<layer>-dev-skeleton`, blockedBy: `[repo-steward's task]`. Description: *"Build the thinnest possible `<layer>` slice that supports the acceptance items touching your folder. Stub everything not yet needed. Commit incrementally. If you hit a cross-folder question you can't answer from `docs/architecture.md`, raise a `blockedBy` on this task referencing a follow-up question task and SendMessage architect with the question. Mark this task `completed` once your slice is in."*
- **Architect** — owner: `architect`, blockedBy: none (the architect is on-call, not gated). Description: *"Be available for cross-folder questions from `<layer>-dev-skeleton` via SendMessage. When the developer's task transitions to `completed`, review the diff and either approve (a follow-up task you raise + mark `completed`) or request changes (a follow-up `FIX:` task you assign to the developer). Mark this task `completed` once the layer is approved."*

After the developer's task and Architect's approval task both transition to `completed`, the poll loop spawns Verifier and opens:

- **`verifier-skeleton-<layer>`** — owner: `verifier-skeleton-<layer>`, blockedBy: none. Description: *"Run whatever acceptance items are demonstrable at this slice. Write `docs/demos/<date>-skeleton-<layer>.md`. Ready-to-advance: Yes if the slice is coherent and the items you could check pass; Not yet if a CRITICAL finding blocks. Mark this task `completed` once the demo is committed."*

On Not yet: the lead raises a FIX task assigned to the developer (with `blockedBy: [original developer task]` if the developer needs the prior context); re-runs Verifier after the fix. On Yes: lead opens the close-branch task for RepoSteward. Advance to the next layer.

**When the last layer lands**, the lead opens one more Verifier task with the **full acceptance list**:
- **`verifier-skeleton-complete`** — Description: *"Run the full acceptance list. Write `docs/demos/<date>-skeleton-complete.md`. Mark `completed` once committed."*

That demo's Ready-to-advance gates Stage 4.

### 3.4 Stage 4 — Refine (parallel work)

**One branch per refine pass.**

Team: `jfdi-<surname>-refine-<N>`.

Spawn (in one message, so they run concurrently):
- `architect` (always — available for cross-folder brokering)
- One `<layer>-dev-refine-<N>` **per folder being touched in this pass**
- `repo-steward`

Verifier (`verifier-refine-<N>`) is spawned later — when the poll loop sees every developer task `completed`, the lead spawns Verifier and opens its task.

First-task descriptions (one `TaskCreate` call, multiple tasks):

- **RepoSteward** — owner: `repo-steward`, blockedBy: none. Description: *"Open branch `feature/refine-<N>` from `main`. Mark `completed` once checked out."*
- **Each `<layer>-dev-refine-<N>`** — owner: that developer, blockedBy: `[repo-steward's task]`. Description: *"Implement acceptance items <list>. Stay in your folder. Commit incrementally. For cross-folder coordination, SendMessage architect and raise a `blockedBy` on this task pointing at the question. Mark `completed` once your share is in."*
- **Architect** — owner: `architect`, blockedBy: none. Description: *"Be available for cross-folder questions from developers via SendMessage. When all developer tasks transition to `completed`, mark this task `completed`."*

After every developer task is `completed`:
- **`verifier-refine-<N>`** — owner: that verifier, blockedBy: none. Description: *"Run the full acceptance list. Write `docs/demos/<date>-refine-<N>.md`. Mark `completed` once committed."*

On Not yet: route CRITICALs to the relevant developers; re-run Verifier. On Yes: RepoSteward merges. Increment N and loop.

## 6. Tool-resolution rules for teammate spawning

When the TeamLead calls `Agent` to add a teammate, it specifies the subagent_type (the role), the teammate name, and optionally overrides for `tools`/`disallowed-tools`/`model`. Key rules:

1. **Teammate frontmatter `tools`/`disallowed-tools` is honoured.** `disallowed-tools` is applied first; `tools` is resolved against the remaining pool.
2. **Teammates inherit the lead's permission restrictions.** Keep the TeamLead's denylist empty; push role-specific restrictions to the specialists themselves.
3. **`AskUserQuestion` is harness-blocked for every teammate**, regardless of frontmatter. Any agent whose role requires structured multi-choice questions must run as the main session. This is why TeamLead is the only supported main-session entry point; Vision intake is TeamLead-driven via the relay pattern.
4. **Plugin-shipped agents load from `<CLAUDE_CONFIG_DIR>/plugins/cache/dr-zog-jfdi-agents/<version>/`** — under the bootstrap-generated `./jfdi.sh` launcher this resolves to `./.claude-state/plugins/cache/dr-zog-jfdi-agents/<version>/`, not the source tree. Frontmatter edits take effect on live installs only after a version bump + `/plugin update` (or a dev-mode install). This does NOT apply to the developer agents the Architect mints into `.claude/agents/` — those are project-scoped and live.
5. **The Architect-minted developers are project-scoped subagents.** They live in `.claude/agents/` in the downstream repo. They are addressable as teammates by the TeamLead. They do not benefit from plugin caching; edits take effect on the next spawn.

## 7. Aliveness & stall detection

The primary stall-detection mechanism is the periodic-poll loop's own five-tick rule (§ 3): if `TaskList` shows no state change for ~5 minutes while a task is `in_progress` with empty `blockedBy`, the lead sends one nudging SendMessage to that task's owner. Reset the no-change counter once you act.

### 5.1 Normal idle

A teammate whose task is `completed` is finished. The TeamLead's poll loop notices on the next tick and reacts.

A teammate whose task is `in_progress` is either (a) still working or (b) idle between turns. The harness says idle between turns is normal. Do not ping unless the five-tick threshold has fired.

A teammate whose task is `pending` with empty `blockedBy` and an `owner` set is supposed to be picking the task up via `TaskList` polling. If it hasn't transitioned to `in_progress` within a couple of poll-loop ticks, the same five-tick stall rule applies.

### 5.2 The stall-detector script (complementary observer)

`${CLAUDE_PLUGIN_ROOT}/scripts/stall-detector.sh` watches the team's filesystem-state directory and reports teammates that have stopped producing activity. The TeamLead may launch it in the background at team creation:

```bash
"${CLAUDE_PLUGIN_ROOT}/scripts/stall-detector.sh" "<team-name>" &
```

Treat its `STALL_DETECTED` lines as a **secondary corroborating signal** alongside the poll loop's own stall detection — not as the primary trigger. If the script fires before the loop's five-tick threshold, that's useful early warning: read the report, but stay with the loop's response (one nudge to the owner, escalate to human only if the loop's subsequent ticks confirm continued silence).

### 5.3 The team-inspect binary

`${CLAUDE_PLUGIN_ROOT}/bin/team-inspect` is a Go binary that reads the Claude Code state directories and produces a structured snapshot of every team, task, and inbox. The TeamLead invokes it when debugging: *"what state is everyone in right now?"* See `tools/team-inspect/README.md` in the repo for usage.

## 8. Status block format

Every stage transition, the TeamLead prints a short block to the human. Format:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Stage: <n> — <name>
Status: <in-progress | complete | blocked>
Team: jfdi-<surname>-<stage>
Teammates: <name1>, <name2>, ...
Last event: <one-line summary>
Next action: <what happens now or what's awaiting>
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

A human returning after a break should be able to read three blocks and know exactly what's going on.

## 9. Checkpoint decisions

**Checkpointed mode (default).** At every load-bearing transition, the TeamLead shows the status block and asks via `AskUserQuestion`: *"Proceed to <next stage>?"* — with choices *Yes*, *Stop here*, *Redirect (free-text)*. The load-bearing transitions are:

- Post-intake (before architecture)
- Post-architecture (before Build starts)
- Each Build-layer demo (before advancing to the next layer)
- Post-skeleton-complete (before Refine starts)
- Each Refine-pass demo (before the next pass)

**Autonomous JFDI.** Same transitions, but no `AskUserQuestion`. The TeamLead proceeds automatically unless Verifier returns Ready-to-advance: Not yet (which routes to a fix loop, never silently pushes past). The human can interrupt at any time.

## 10. Failure recovery playbook

### 8.1 A minted developer agent is malformed

Symptom: TeamLead attempts to spawn `<layer>-dev-...`, `Agent` call fails with *"subagent definition not found"* or parse error.

Resolution: The Architect minted the agent badly. Route back to Architect as a `FIX:` task quoting the error. Do not attempt to hand-edit `.claude/agents/` — regenerate via the `write-agent` skill.

### 8.2 A developer touches files outside its folder

Symptom: Verifier's demo has a CRITICAL finding citing cross-folder writes.

Resolution: Two paths depending on intent.
- **Developer made a mistake.** Route a `FIX:` task to the developer: *"Revert your changes in `<other-folder>/`. If you think you need something from that folder, ask Architect via SendMessage."*
- **The folder map needs updating.** If the Architect judges the cross-folder write was necessary, the folder map is wrong. Route back to Architect: *"Revise `docs/architecture.md` folder map and re-mint the affected developers via the write-agent skill."*

### 8.3 A technical dispute reaches the Architect

Flow: Architect reads both positions, writes the ruling to `docs/decisions.md`, SendMessages both parties with the ruling. Parties are bound. If one party pushes back, Architect re-reads, revises or confirms, commits the decision to the log. Second appeal is to the human.

### 8.4 Verifier finds the system won't start

Symptom: Demo has `Ready-to-advance: Not yet`, startup CRITICAL.

Resolution: Route to the developer of the layer Verifier's log implicates. If it's a cross-layer issue, route to Architect first for triage. Do not advance until a subsequent demo shows the system starts.

### 8.5 Zombie team recovery (destructive — read twice)

Symptom: a team directory exists under `$CLAUDE_CONFIG_DIR/teams/<name>/` (or `~/.claude/teams/<name>/` if `CLAUDE_CONFIG_DIR` is unset) with no recent file activity; `SendMessage` writes succeed but no teammate ever responds.

**Before you run any `rm`:**

1. **Confirm the team name matches this session's surname.** The TeamLead picked a surname at session start (see § 1.1) and every team this session created follows the `jfdi-<surname>-*` pattern. If the zombie directory's name does not start with the surname this session is using, **STOP** — you are about to destroy another project's team state. Escalate to the human via `AskUserQuestion`: *"I see a zombie team `<name>` that does not match this session's surname `<our-surname>`. It likely belongs to another project. Should I leave it alone?"*
2. **Snapshot `config.json`'s `members` array** so you can re-spawn the same teammates under the same names.
3. **Remove both directories together**, scoped to the verified name:
   ```bash
   STATE_DIR="${CLAUDE_CONFIG_DIR:-$HOME/.claude}"
   rm -rf "$STATE_DIR/teams/<verified-name>" "$STATE_DIR/tasks/<verified-name>"
   ```
4. **`TeamCreate` with the same name and re-spawn** each teammate from the snapshot.

**Never improvise the `rm`.** A wildcard, a guessed name, or a "clean everything" sweep is always wrong. One verified team name per recovery.

## 11. Disallowed behaviours

These behaviours deadlocked earlier eval runs and are explicitly forbidden:

- **Spawning two developers before the walking skeleton exists.** Sequential-skeleton rule.
- **Silently tolerating cross-folder writes.** Escalate to Architect.
- **Proceeding past a Not-yet demo.** The gate is hard.
- **Editing content files yourself.** The TeamLead is read-only. Route writes to the right specialist.
- **Pinging an idle teammate.** Idle between turns is normal per harness docs. Only nudge when the poll loop's five-tick stall threshold has fired.
- **Sending work briefs or state transitions via SendMessage.** Work briefs go in `TaskCreate.description`; state transitions go via `TaskUpdate`. SendMessage is for nudges and out-of-band Q&A only (see § 2).
- **Spawn prompts that contain a work brief.** Spawn = role orientation only (see § 2's spawn-prompt template). All actionable content goes in the task description.
- **Sending `SendMessage({to: "TeamLead", ...})` from a teammate.** The lead's addressable name is `team-lead`, lowercase kebab-case. Using the role label (`TeamLead`) silently dead-letters.
