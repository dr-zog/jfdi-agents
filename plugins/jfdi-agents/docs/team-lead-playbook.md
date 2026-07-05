# TeamLead playbook

> Operational mechanics for the TeamLead role. The roster (`roster.md`) says what the TeamLead's scope is. The process doc (`process.md`) says what the overall workflow is. This document is the TeamLead's how-to: naming conventions, checkpoint cadence, stall detection, the status block format shown to the human, the exact sequence of tool calls at each stage boundary.

## 1. Conventions

### 1.1 Team name — chosen by the harness

As of Claude Code v2.1.178, `TeamCreate` and `TeamDelete` are removed and there is **exactly one team per session**, formed automatically when the first teammate is spawned. The harness names it `session-<first-8-of-session-id>`. The TeamLead does not pick a team name; any `team_name` passed to the `Agent` tool is accepted but ignored.

The team is cleaned up automatically when the session ends. Do not attempt to tear it down mid-session — there is nothing to tear down that stays behind for the next stage; the same team runs throughout.

### 1.2 Teammate naming

All teammate names are lowercase kebab-case. Role name first, suffix last:

| Role | Teammate name pattern | Example |
|---|---|---|
| ProductOwner | `product-owner` | `product-owner` |
| Architect | `architect` | `architect` |
| Developer (minted) | `<layer>-dev-<phase>-<suffix>` | `backend-dev-skeleton`, `frontend-dev-refine-3` |
| Verifier | `verifier-<phase>` | `verifier-skeleton-complete` (Stage 3 end), `verifier-refine-1` (Stage 4 pass N) |
| RepoSteward | `repo-steward` | `repo-steward` (one per session) |

Naming rules:
- Lowercase kebab-case. No underscores, no camelCase.
- The phase suffix makes it obvious which slice of work a teammate belongs to (Build-phase vs Refine-pass-N).
- A role may be spawned more than once during the project lifetime (a new Refine pass mints fresh developer teammates) — each spawn gets a fresh suffix.
- Persistent roles (`product-owner`, `architect`, `repo-steward`) get **one spawn per session** and stay throughout.

### 1.3 Commit-author slugs

Whatever the teammate name is, that's also the author slug: `<teammate-name>@jfdi-agents.invalid`. So `backend-dev-skeleton`'s commits have author `backend-dev-skeleton <backend-dev-skeleton@jfdi-agents.invalid>`. See `process.md` § "Commit authorship".

### 1.4 Branch naming

| Stage | Branch name |
|---|---|
| Vision intake | `feature/vision` |
| Architecture & team design | `feature/architecture` |
| Build (walking skeleton) | `feature/skeleton` (one branch for the whole DAG chain — all layer devs commit on this branch under the DAG-up-front model) |
| Refine — per pass | `feature/refine-<N>` (e.g. `feature/refine-1`) |

One branch per stage. RepoSteward creates at stage start, merges at stage close.

## 2. The TeamLead's two channels — Tasks for state, SendMessage for nudges

This split is the single most important operational rule for the TeamLead. It is also covered in `${CLAUDE_PLUGIN_ROOT}/docs/process.md` § "The two channels"; this section restates it from the lead's perspective.

**Tasks are the work-and-state channel.** All work you assign goes into `TaskCreate` with `owner`, `blockedBy`, `status: pending`, and a self-contained description. All teammate progress comes back to you via `TaskUpdate` — `in_progress` when an agent picks up, `completed` when it finishes, new `blockedBy` entries when it hits a dependency. You read this state via the periodic-poll loop (§ "The periodic-poll loop"). The Task channel is the source of truth.

**SendMessage is the nudge-and-out-of-band-Q&A channel.** You use it for:

- **Stall-check nudges** — when the periodic-poll loop's five-tick (~10-minute) threshold fires, SendMessage the owner of the stale `in_progress` task asking what is blocking them. One sentence. No rich brief.
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

The one legitimate wake-style SendMessage is the stall-check nudge: *"team-lead → backend-dev: task #7 has been in_progress for 10 minutes with no updates. What's blocking you?"* One sentence. No work brief.

### Idle notifications are not events

A bare idle notification from a teammate is not an event. Zero response, zero state-check. Every teammate emits an idle notification after every turn — that is the harness's design, not a signal that something needs your attention. Ignore them. The poll loop (§ 3) is what drives you. Full statement of the rule in `${CLAUDE_PLUGIN_ROOT}/agents/team-lead.md`.

### Never ask, read

Status comes from disk (`TaskList`, `TaskGet`, `git log`, `git status`), not from asking a teammate what it is doing. Do not SendMessage teammates for status. Two failure modes this prevents: (1) messages cross in flight with the artefacts they ask about, so the reply is moot before it arrives; (2) the question invites the teammate to respond with prose, waking you and consuming context for information that was on disk. Inbound SendMessages from teammates are **confirmation** of already-observable state, not **triggers** for TeamLead action. Full statement of the rule in `${CLAUDE_PLUGIN_ROOT}/agents/team-lead.md`.

### Choreography vs quality gates

The choreography — polling cadence, spawn timing, whether you nudge or wait — is soft. Trust the agents; slacken your ceremony. The quality gates — sequential-skeleton rule, folder-ownership, no-advance-past-Not-yet, Verifier sign-off, `blockedBy` authoritative — are non-negotiable. Slackening one is fine; slackening the other is a bug. Full statement in `${CLAUDE_PLUGIN_ROOT}/agents/team-lead.md` § "Choreography is discretionary; quality gates are non-negotiable".

## 3. The periodic-poll loop

Authoritative version lives in `${CLAUDE_PLUGIN_ROOT}/agents/team-lead.md` § "The periodic-poll loop". Summary for cross-reference:

- **Mechanism: `ScheduleWakeup`** (the harness's `/loop`-dynamic interface). Each tick the TeamLead does its `TaskList` + reactions, then before yielding the turn calls `ScheduleWakeup(delaySeconds: 120)` so the harness re-enters the lead for the next tick. **Not `Bash(sleep)`** — that holds one turn open across an entire stage, balloons context, and is un-interruptible.
- Every ~120 seconds, `TaskList` the team's task state. (120s sits inside the prompt-cache window, so each wake stays warm.)
- React to state changes — clear `blockedBy` entries when blockers complete, spawn follow-up specialists when their cue task completes, close the stage when all tasks are `completed`.
- Five consecutive ticks (~10 minutes) with no state change AND a task is `in_progress` ⇒ stall threshold fires ⇒ SendMessage the owner asking what is blocking them. One specific teammate, one sentence.
- **Idle-after-spawn is NOT a stall signal.** A freshly spawned teammate goes idle until its first poll picks up its task; that is the harness confirming registration, not a problem. The stall threshold is the only nudge trigger. (Authoritative version in `${CLAUDE_PLUGIN_ROOT}/agents/team-lead.md` § "The periodic-poll loop".)
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

4. **Ask the human to confirm.** Via `AskUserQuestion`, confirm (a) the stage you've diagnosed is correct, (b) they want to proceed in Checkpointed or Autonomous JFDI mode.
5. **Spawn the teammates the stage needs.** See § 5 below. The first spawn forms the team implicitly; the harness names it `session-<first-8-of-session-id>`. No `TeamCreate` call.

## 5. Per-stage spawn playbook

Under the single-session-team model, each stage transition **adds new teammates** to the running team. Persistent roles (`product-owner`, `architect`, `repo-steward`) are spawned once and stay for the whole project; ephemeral roles (per-layer skeleton devs, per-phase verifiers, per-pass refine devs) are spawned as their turn comes and can be shutdown-requested when their work is done.

### 5.1 Stage 1 — Intake

**Add to team (first spawns — this implicitly forms the team):**
- `product-owner` (the interactive interview)
- `repo-steward`

First-task descriptions (paste these into `TaskCreate.description` — spawn prompts stay role-orientation-only per § 2):

- **RepoSteward** — owner: `repo-steward`, blockedBy: none. Description: *"Open branch `feature/vision` from `main`. Mark this task `completed` once the branch is checked out."*
- **ProductOwner** — owner: `product-owner`, blockedBy: `[repo-steward's task above]`. Description: *"Run the Vision intake interview. Ask one question at a time via SendMessage to team-lead; team-lead will relay to the human via AskUserQuestion. Produce the five files in `vision/` described in the roster. Do not write `vision/acceptance.md` yet — that happens in Stage 2. Mark this task `completed` once the five files are committed."*

At stage close, the poll loop notices both tasks complete; TeamLead opens a final close-branch task for RepoSteward and moves on to Stage 2 (no team teardown — the same team continues).

### 5.2 Stage 2 — Architecture & team design

**Add to team:**
- `architect` (new — the technical authority)

**Already on team from Stage 1** (no re-spawn):
- `product-owner` (co-authors the acceptance list)
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

### 5.3 Stage 3 — Build (walking skeleton, DAG-up-front)

**One branch, one persistent team, one task graph created up front.** Every layer developer and the skeleton-complete Verifier are spawned at Stage 3 start; every task is created in one `TaskCreate` call with `blockedBy` chains encoding the layer dependency graph. The harness auto-unblocks downstream tasks as their blockers complete; the TeamLead supervises by exception via the periodic-poll loop.

**Add to team at Stage 3 start (all at once):**
- One `<layer>-dev-skeleton` per layer in the Architect's Developer roster (e.g. `shared-dev-skeleton`, `data-dev-skeleton`, `backend-dev-skeleton`, `frontend-dev-skeleton`). Each spawned via `Agent(subagent_type: "<layer>-dev", prompt: "role-orientation")` — never by inlining the body.
- `verifier-skeleton-complete` — spawned upfront, its task will unblock when the last Architect ratification completes.

**Already on team** (no re-spawn):
- `architect`, `product-owner`, `repo-steward` (persistent since earlier stages).

**Canonical task graph** (using layer order `shared → data → backend → frontend`; adapt to whichever layers and order the Architect chose):

| # | Owner | blockedBy | Description (Architect authors these upfront, extracted from `docs/architecture.md`) |
|---|---|---|---|
| 1 | `repo-steward` | — | *"Open branch `feature/skeleton` from `main`. Mark `completed` once checked out."* |
| 2 | `shared-dev-skeleton` | [1] | *"Build the thinnest possible `shared/` slice that supports the acceptance items your folder needs to satisfy. Contracts your layer provides to backend: `<list>`. Stub anything not yet needed. Commit incrementally. Cross-folder questions → raise `blockedBy` + SendMessage architect. Mark `completed` once your slice is in."* |
| 3 | `architect` | [2] | *"Ratify shared's skeleton slice. Review the diff against `docs/architecture.md`'s shared-layer contract. If correct, mark `completed` — this unblocks data-dev. If needs changes, raise a FIX task assigned to `shared-dev-skeleton`; do NOT mark this task `completed` until the FIX is in."* |
| 4 | `data-dev-skeleton` | [3] | *"Build data slice. Contracts: `<from-shared>`, `<to-backend>`. Rest same as shared task."* |
| 5 | `architect` | [4] | *"Ratify data's slice. Same procedure as task 3."* |
| 6 | `backend-dev-skeleton` | [5] | *"Build backend slice. Contracts: `<from-data>`, `<to-frontend>`. Rest same."* |
| 7 | `architect` | [6] | *"Ratify backend's slice."* |
| 8 | `frontend-dev-skeleton` | [7] | *"Build frontend slice. Contracts: `<from-backend>`. Rest same."* |
| 9 | `architect` | [8] | *"Ratify frontend's slice."* |
| 10 | `verifier-skeleton-complete` | [9] | *"Run the FULL acceptance list end-to-end against the walking skeleton. Write `docs/demos/<date>-skeleton-complete.md`. Ready-to-advance: Yes if every item passes; Not yet if a CRITICAL finding blocks. Mark `completed` once the demo is committed."* |
| 11 | `repo-steward` | [10] | *"Close branch `feature/skeleton` — merge to `main`, delete the branch."* |

**Under DAG-up-front there are no per-layer Verifier tasks.** Architect ratification is the per-layer quality gate; Verifier's independent audit runs once at the end against the full acceptance list. This is a deliberate simplification from the pre-#20 model — see the "Why per-layer Verifier tasks are absent from the DAG" note in `${CLAUDE_PLUGIN_ROOT}/agents/team-lead.md`.

**How work flows.** shared-dev picks up task 2 from its own `TaskList` polling (task 1 has just completed); works; marks `completed`. The harness auto-clears task 3's `blockedBy` — Architect picks it up, reviews, marks `completed`. That auto-clears task 4's `blockedBy` — data-dev picks it up. And so on. The TeamLead does not send "task N is now unblocked" SendMessages — the harness does the unblocking; the developer's polling does the pickup.

**Failure-mode routing.**
- **A layer fails Architect ratification.** Architect raises a FIX task assigned to the failing developer, with `blockedBy` on the original dev task. The dev picks up FIX; ratification task 3 stays incomplete; downstream tasks stay gated. When FIX completes, Architect can now mark the ratification `completed`.
- **A layer needs a cross-folder contract revision.** The dev raises a `blockedBy` on their own task pointing at a new question task; SendMessages architect. Architect resolves via a decisions.md entry + a follow-up briefing task (or FIX) if the earlier layer needs to change. Downstream chain stays gated naturally.
- **Verifier-skeleton-complete returns `Not yet`.** Raise FIX tasks assigned to the developers whose folders are implicated (blockedBy: the failing Verifier task). Verifier's task stays incomplete. When FIXes land, ratify them via Architect follow-ups, then re-run Verifier by raising a fresh task assigned to `verifier-skeleton-complete`.

**Retiring after Stage 3.** Once task 11 completes (branch merged, deleted), every `<layer>-dev-skeleton` teammate and `verifier-skeleton-complete` have finished their session role. Send each a shutdown request (*"Ask the `<teammate-name>` teammate to shut down — the skeleton is merged and demoed."*). Persistent roles (`architect`, `product-owner`, `repo-steward`) carry into Stage 4.

### 5.4 Stage 4 — Refine (parallel work, DAG-up-front)

**One branch per refine pass, one task graph per pass.** Every developer for the pass and the pass Verifier are spawned at pass start; every task is created in one `TaskCreate` call. Developers run in parallel (no `blockedBy` chain between them); Verifier is blocked by all of them; RepoSteward's close is blocked by Verifier.

**Add to team at pass start (all at once):**
- One `<layer>-dev-refine-<N>` per folder touched in this pass (spawned in one message; each via `Agent(subagent_type: "<layer>-dev", …)`).
- `verifier-refine-<N>`.

**Already on team** (no re-spawn):
- `architect` (on-call for cross-folder brokering — NOT in the chain).
- `product-owner`, `repo-steward` (persistent).

**Canonical task graph:**

| # | Owner | blockedBy | Description |
|---|---|---|---|
| 1 | `repo-steward` | — | *"Open branch `feature/refine-<N>` from `main`. Mark `completed` once checked out."* |
| 2 | `data-dev-refine-<N>` | [1] | *"Implement acceptance items `<list>` that touch data/. Stay in your folder. Commit incrementally. Cross-folder → raise blockedBy + SendMessage architect. Mark `completed` once your share is in."* |
| 3 | `backend-dev-refine-<N>` | [1] | *"…backend items `<list>`…"* |
| 4 | `frontend-dev-refine-<N>` | [1] | *"…frontend items `<list>`…"* |
| 5 | `verifier-refine-<N>` | [2, 3, 4] | *"Run the FULL acceptance list. Write `docs/demos/<date>-refine-<N>.md`. Ready-to-advance: Yes/Not-yet. Mark `completed` once the demo is committed."* |
| 6 | `repo-steward` | [5] | *"Close branch `feature/refine-<N>` — merge to `main`, delete the branch."* |

**Architect is not in the chain.** They persist on the team as the on-call cross-folder authority. If a developer surfaces a cross-folder contract question, they SendMessage `architect` with the prose question and raise a `blockedBy` on their own task; Architect rules via SendMessage + `docs/decisions.md`; developer clears their `blockedBy`. Architect does NOT own ratification tasks in Refine — the parallel model doesn't need them because Verifier is the gate at pass end.

**Failure-mode routing.**
- **A dev raises a cross-folder blockedBy.** Route to Architect (developer already SendMessaged); Architect resolves; developer clears blockedBy and resumes.
- **Verifier returns `Not yet`.** FIX tasks assigned to the implicated developers, blockedBy on the failing Verifier task. Verifier task stays incomplete. When FIXes land, re-run Verifier via a fresh task.

**Retiring after a Refine pass.** Once task 6 completes (pass branch merged), every `<layer>-dev-refine-<N>` teammate and `verifier-refine-<N>` have finished their role for this pass. Send each a shutdown request. The next pass mints fresh `<layer>-dev-refine-<N+1>` teammates.

Loop: increment N and repeat until the acceptance list is fully green or the human halts.

## 6. Tool-resolution rules for teammate spawning

When the TeamLead calls `Agent` to add a teammate, it specifies the subagent_type (the role), a teammate name, and optionally overrides for `tools`/`disallowed-tools`/`model`. Key rules:

1. **Always spawn via `subagent_type`, never by passing the full agent body as the `prompt`.** The prompt argument to `Agent` is meant for role-orientation only — one paragraph naming the role, the team, and the fact that the teammate will pick up assigned tasks via `TaskList`. If you pass the full body text of a subagent as the prompt, the harness treats the teammate as an ad-hoc spawn: it carries no role metadata, appears in the exit menu as raw prompt text ("You are verifier-refine-3, the independent Verifier…") instead of structured metadata ("Role: Verifier. Team: session (…)"), and cannot benefit from later subagent-definition edits. Both the plugin-shipped specialists and the Architect-minted `.claude/agents/<layer>-dev.md` files are proper subagent definitions — reference them by `subagent_type`, not by inlining their bodies.
2. **Teammate frontmatter `tools`/`disallowed-tools` is honoured.** `disallowed-tools` is applied first; `tools` is resolved against the remaining pool.
3. **Teammates inherit the lead's permission restrictions.** Keep the TeamLead's denylist empty; push role-specific restrictions to the specialists themselves.
4. **`AskUserQuestion` is harness-blocked for every teammate**, regardless of frontmatter. Any agent whose role requires structured multi-choice questions must run as the main session. This is why TeamLead is the only supported main-session entry point; Vision intake is TeamLead-driven via the relay pattern.
5. **Plugin-shipped agents load from `<CLAUDE_CONFIG_DIR>/plugins/cache/dr-zog-jfdi-agents/<version>/`** — under the bootstrap-generated `./jfdi.sh` launcher this resolves to `./.claude-state/plugins/cache/dr-zog-jfdi-agents/<version>/`, not the source tree. Frontmatter edits take effect on live installs only after a version bump + `/plugin update` (or a dev-mode install). This does NOT apply to the developer agents the Architect mints into `.claude/agents/` — those are project-scoped and live.
6. **The Architect-minted developers are project-scoped subagents.** They live in `.claude/agents/` in the downstream repo. They are addressable as teammates by the TeamLead via `subagent_type: "<layer>-dev"`. They do not benefit from plugin caching; edits take effect on the next spawn.
7. **`team_name` on the `Agent` tool is accepted but ignored** (per the harness docs, as of v2.1.178). The team is fixed as `session-<first-8-of-session-id>`; don't try to override it.

## 7. Aliveness & stall detection

The primary stall-detection mechanism is the periodic-poll loop's own five-tick rule (§ 3): if `TaskList` shows no state change for ~10 minutes while a task is `in_progress` with empty `blockedBy`, the lead sends one nudging SendMessage to that task's owner. Reset the no-change counter once you act.

### 7.1 Normal idle

A teammate whose task is `completed` is finished. The TeamLead's poll loop notices on the next tick and reacts.

A teammate whose task is `in_progress` is either (a) still working or (b) idle between turns. The harness says idle between turns is normal. Do not ping unless the five-tick threshold has fired.

A teammate whose task is `pending` with empty `blockedBy` and an `owner` set is supposed to be picking the task up via `TaskList` polling. If it hasn't transitioned to `in_progress` within a couple of poll-loop ticks, the same five-tick stall rule applies.

**A freshly spawned teammate goes idle immediately and stays that way until its next poll picks up its task.** That is the harness confirming registration, not a stall — do not nudge. The five-tick threshold (~10 minutes) is the only legitimate stall trigger.

### 7.2 The stall-detector script (complementary observer)

`${CLAUDE_PLUGIN_ROOT}/scripts/stall-detector.sh` watches the team's filesystem-state directory and reports teammates that have stopped producing activity. The TeamLead may launch it in the background at team creation:

```bash
"${CLAUDE_PLUGIN_ROOT}/scripts/stall-detector.sh" "<team-name>" &
```

Treat its `STALL_DETECTED` lines as a **secondary corroborating signal** alongside the poll loop's own stall detection — not as the primary trigger. If the script fires before the loop's five-tick threshold, that's useful early warning: read the report, but stay with the loop's response (one nudge to the owner, escalate to human only if the loop's subsequent ticks confirm continued silence).

### 7.3 The team-inspect binary

`${CLAUDE_PLUGIN_ROOT}/bin/team-inspect` is a Go binary that reads the Claude Code state directories and produces a structured snapshot of every team, task, and inbox. The TeamLead invokes it when debugging: *"what state is everyone in right now?"* See `tools/team-inspect/README.md` in the repo for usage.

## 8. Status block format

Every stage transition, the TeamLead prints a short block to the human. Format:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Stage: <n> — <name>
Status: <in-progress | complete | blocked>
Team: session-<hash>  (harness-derived, see § 1.1)
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

### 10.1 A minted developer agent is malformed

Symptom: TeamLead attempts to spawn `<layer>-dev-...`, `Agent` call fails with *"subagent definition not found"* or parse error.

Resolution: The Architect minted the agent badly. Route back to Architect as a `FIX:` task quoting the error. Do not attempt to hand-edit `.claude/agents/` — regenerate via the `write-agent` skill.

### 10.2 A developer touches files outside its folder

Symptom: Verifier's demo has a CRITICAL finding citing cross-folder writes.

Resolution: Two paths depending on intent.
- **Developer made a mistake.** Route a `FIX:` task to the developer: *"Revert your changes in `<other-folder>/`. If you think you need something from that folder, ask Architect via SendMessage."*
- **The folder map needs updating.** If the Architect judges the cross-folder write was necessary, the folder map is wrong. Route back to Architect: *"Revise `docs/architecture.md` folder map and re-mint the affected developers via the write-agent skill."*

### 10.3 A technical dispute reaches the Architect

Flow: Architect reads both positions, writes the ruling to `docs/decisions.md`, SendMessages both parties with the ruling. Parties are bound. If one party pushes back, Architect re-reads, revises or confirms, commits the decision to the log. Second appeal is to the human.

### 10.4 Verifier finds the system won't start

Symptom: Demo has `Ready-to-advance: Not yet`, startup CRITICAL.

Resolution: Route to the developer of the layer Verifier's log implicates. If it's a cross-layer issue, route to Architect first for triage. Do not advance until a subsequent demo shows the system starts.

### 10.5 Resumption after a session restart or crash

Under the single-session-team model, there is no "zombie team recovery" scenario in the old sense — the harness auto-cleans a team's config directory at session exit, and each new session gets a fresh `session-<hash>` team. The scenarios that DO arise:

**Resumed session (`/resume` or `/rewind`).** The harness restores the task list but **does not restore in-process teammates** (per the agent-teams docs: *"the lead may attempt to message teammates that no longer exist"*). Recovery:

1. **Read `TaskList`** — this is your ground truth for where the prior session left off.
2. **Read the on-disk state** — `vision/`, `docs/architecture.md`, `.claude/agents/*-dev.md`, `docs/demos/`, `git log`.
3. **Re-spawn only the teammates you need for the next work** — the persistent roles (`architect`, `product-owner`, `repo-steward`) plus whatever ephemeral role owns the currently-`in_progress` or next task. Do NOT try to re-spawn every teammate the prior session had; the ephemeral ones (e.g. `verifier-skeleton-complete`) have their work already committed.

**Stale task directory from a killed session.** If the prior session was killed without clean exit (crash, kill -9, host reboot), the harness's cleanup may not have fired. You'll see a `tasks/session-<old-hash>/` directory that does not match this session's hash. Do NOT try to clean it up yourself — surface it to the human via `AskUserQuestion` as an operator hygiene item. The `${CLAUDE_PLUGIN_ROOT}/bin/team-inspect` tool can characterise it if useful.

**Never run `rm -rf` on `<CLAUDE_CONFIG_DIR>/teams/` or `<CLAUDE_CONFIG_DIR>/tasks/`.** The old per-project `rm` recovery routine assumed multiple teams per project; that assumption is gone. If a directory looks stale, ask the human — they own destructive cleanup.

## 11. Disallowed behaviours

These behaviours deadlocked earlier eval runs and are explicitly forbidden:

- **Letting two developers do skeleton *work* in parallel.** Under DAG-up-front (§ 5.3) all skeleton devs are spawned at once, but their tasks are chained via `blockedBy` so only one is actively working at a time. Skipping the chain (creating unblocked parallel skeleton tasks) violates the sequential-skeleton rule.
- **Silently tolerating cross-folder writes.** Escalate to Architect.
- **Proceeding past a Not-yet demo.** The gate is hard.
- **Editing content files yourself.** The TeamLead is read-only. Route writes to the right specialist.
- **Pinging an idle teammate.** Idle between turns is normal per harness docs. Only nudge when the poll loop's five-tick stall threshold has fired.
- **Sending work briefs or state transitions via SendMessage.** Work briefs go in `TaskCreate.description`; state transitions go via `TaskUpdate`. SendMessage is for nudges and out-of-band Q&A only (see § 2).
- **Spawn prompts that contain a work brief.** Spawn = role orientation only (see § 2's spawn-prompt template). All actionable content goes in the task description.
- **Sending `SendMessage({to: "TeamLead", ...})` from a teammate.** The lead's addressable name is `team-lead`, lowercase kebab-case. Using the role label (`TeamLead`) silently dead-letters.
