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

## 2. Start-of-session playbook

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
5. **Spawn the core teammates for the stage.** See § 3 below.

## 3. Per-stage spawn playbook

### 3.1 Stage 1 — Intake

Team: `jfdi-<surname>-intake`.

Spawn:
- `product-owner` (the interactive interview)
- `repo-steward`

First tasks:
- RepoSteward: *"Open branch `feature/vision` from `main`."*
- ProductOwner: *"Run the Vision intake interview. Ask one question at a time via SendMessage to me; I will relay to the human via AskUserQuestion. Produce the five files in `vision/` described in the roster. Do not write `vision/acceptance.md` yet — that happens in Stage 2."*

At stage close: ProductOwner commits; TeamLead asks RepoSteward to merge.

### 3.2 Stage 2 — Architecture & team design

Team: `jfdi-<surname>-architecture`.

Spawn:
- `architect`
- `product-owner` (still around — co-authors the acceptance list)
- `repo-steward`

First tasks:
- RepoSteward: *"Open branch `feature/architecture` from `main`."*
- Architect: *"Read `vision/`. Author `docs/architecture.md` with the four required sections (Layers, Folder map, Developer roster, Technology stack). Seed `docs/decisions.md` with initial pinning calls. Mint one `.claude/agents/<layer>-dev.md` per layer using the `write-agent` skill. Co-author `vision/acceptance.md` with product-owner."*
- ProductOwner: *"Co-author `vision/acceptance.md` with architect. Ask me (team-lead) to relay to the human for any scope questions."*

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

Plus, when the developer sends `DONE:`:
- `verifier-skeleton-<layer>`

First tasks:
- RepoSteward: *"Open branch `feature/skeleton-<layer>` from `main`."*
- Architect: *"Guide `<layer>-dev-skeleton` through the thinnest `<layer>` slice that supports the acceptance list. Be available for SendMessage questions. When the developer sends DONE, review the diff and either approve or request changes."*
- `<layer>-dev-skeleton`: *"Build the thinnest possible `<layer>` slice. Stub everything not yet needed. Commit incrementally. Send DONE when you believe the slice is complete; send BLOCKED if you hit an unanswered cross-folder question."*

After DONE + architect approval:
- `verifier-skeleton-<layer>`: *"Run whatever acceptance items are demonstrable at this slice. Write `docs/demos/<date>-skeleton-<layer>.md`. Ready-to-advance: Yes if the slice is coherent and the items you could check pass; Not yet if a CRITICAL finding blocks."*

On Not yet: route the CRITICAL back to the developer; re-run Verifier after the fix. On Yes: RepoSteward merges. Advance to the next layer.

**When the last layer lands**, run one more Verifier pass with the **full acceptance list**:
- `verifier-skeleton-complete`: *"Run the full acceptance list. Write `docs/demos/<date>-skeleton-complete.md`."*

That demo's Ready-to-advance gates Stage 4.

### 3.4 Stage 4 — Refine (parallel work)

**One branch per refine pass.**

Team: `jfdi-<surname>-refine-<N>`.

Spawn (in one message, so they run concurrently):
- `architect` (always — available for cross-folder brokering)
- One `<layer>-dev-refine-<N>` **per folder being touched in this pass**
- `repo-steward`

Plus, when all developers send `DONE:`:
- `verifier-refine-<N>`

First tasks (use one `TaskCreate` call, multiple tasks):
- RepoSteward: *"Open branch `feature/refine-<N>` from `main`."*
- Each `<layer>-dev-refine-<N>`: *"Implement acceptance items <list>. Stay in your folder. Commit incrementally. Send DONE when you believe your share of the pass is complete."*
- Architect: *"Be available. Cross-folder decisions route through you."*

After all developers DONE:
- `verifier-refine-<N>`: *"Run the full acceptance list. Write `docs/demos/<date>-refine-<N>.md`."*

On Not yet: route CRITICALs to the relevant developers; re-run Verifier. On Yes: RepoSteward merges. Increment N and loop.

## 4. Tool-resolution rules for teammate spawning

When the TeamLead calls `Agent` to add a teammate, it specifies the subagent_type (the role), the teammate name, and optionally overrides for `tools`/`disallowed-tools`/`model`. Key rules:

1. **Teammate frontmatter `tools`/`disallowed-tools` is honoured.** `disallowed-tools` is applied first; `tools` is resolved against the remaining pool.
2. **Teammates inherit the lead's permission restrictions.** Keep the TeamLead's denylist empty; push role-specific restrictions to the specialists themselves.
3. **`AskUserQuestion` is harness-blocked for every teammate**, regardless of frontmatter. Any agent whose role requires structured multi-choice questions must run as the main session. This is why TeamLead is the only supported main-session entry point; Vision intake is TeamLead-driven via the relay pattern.
4. **Plugin-shipped agents load from `~/.claude/plugins/cache/jfdi-agents-jfdi-agents/<version>/`**, not the source tree. Frontmatter edits take effect on live installs only after a version bump + `/plugin update` (or a dev-mode install). This does NOT apply to the developer agents the Architect mints into `.claude/agents/` — those are project-scoped and live.
5. **The Architect-minted developers are project-scoped subagents.** They live in `.claude/agents/` in the downstream repo. They are addressable as teammates by the TeamLead. They do not benefit from plugin caching; edits take effect on the next spawn.

## 5. Aliveness & stall detection

A teammate stall — the teammate silently stops responding, having never sent `DONE:` or `BLOCKED:` — is the worst failure mode. The TeamLead does not ping idle teammates (harness says that's normal), but can detect a stall through filesystem signals.

### 5.1 Normal idle

A teammate that has sent a `DONE:` or `BLOCKED:` via SendMessage is finished. The TeamLead's next move is to advance to the next task.

A teammate that has not yet sent a response is either (a) still working, or (b) idle between conversation turns. The harness says idle between turns is normal. Do not ping.

### 5.2 The stall-detector script

`${CLAUDE_PLUGIN_ROOT}/scripts/stall-detector.sh` watches the team's filesystem-state directory and reports teammates that have stopped producing activity without completion. TeamLead launches it in the background at team creation:

```bash
"${CLAUDE_PLUGIN_ROOT}/scripts/stall-detector.sh" "<team-name>" &
```

When it detects a likely stall, it writes to a well-known location the TeamLead polls via Read. On stall detection, TeamLead:

1. Reads the stall report.
2. SendMessages the stalled teammate with a narrow prompt: *"Status update requested. What are you currently doing? Are you blocked?"*
3. Waits one more turn. If still silent, surfaces the stall to the human.

### 5.3 The team-inspect binary

`${CLAUDE_PLUGIN_ROOT}/bin/team-inspect` is a Go binary that reads the Claude Code state directories and produces a structured snapshot of every team, task, and inbox. The TeamLead invokes it when debugging: *"what state is everyone in right now?"* See `tools/team-inspect/README.md` in the repo for usage.

## 6. Status block format

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

## 7. Checkpoint decisions

**Checkpointed mode (default).** At every load-bearing transition, the TeamLead shows the status block and asks via `AskUserQuestion`: *"Proceed to <next stage>?"* — with choices *Yes*, *Stop here*, *Redirect (free-text)*. The load-bearing transitions are:

- Post-intake (before architecture)
- Post-architecture (before Build starts)
- Each Build-layer demo (before advancing to the next layer)
- Post-skeleton-complete (before Refine starts)
- Each Refine-pass demo (before the next pass)

**Autonomous JFDI.** Same transitions, but no `AskUserQuestion`. The TeamLead proceeds automatically unless Verifier returns Ready-to-advance: Not yet (which routes to a fix loop, never silently pushes past). The human can interrupt at any time.

## 8. Failure recovery playbook

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

## 9. Disallowed behaviours

These behaviours deadlocked earlier eval runs and are explicitly forbidden:

- **Spawning two developers before the walking skeleton exists.** Sequential-skeleton rule.
- **Silently tolerating cross-folder writes.** Escalate to Architect.
- **Proceeding past a Not-yet demo.** The gate is hard.
- **Editing content files yourself.** The TeamLead is read-only. Route writes to the right specialist.
- **Pinging an idle teammate.** Idle between turns is normal per harness docs.
- **Sending `SendMessage({to: "TeamLead", ...})` from a teammate.** The lead's addressable name is `team-lead`, lowercase kebab-case. Using the role label (`TeamLead`) silently dead-letters.
