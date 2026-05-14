# Process

> This document is the team's operating manual. It describes how sessions start, how agents hand off, how conflicts escalate, how quality gates work, and how the plugin wires into the [Claude Code agent team](https://code.claude.com/docs/en/agent-teams) machinery. Every agent's system prompt refers to this document; when in doubt, defer to this file.

## Scope of this document

This file answers six questions:

1. **Who runs first?** — the canonical session order for a new project.
2. **How does an agent hand off to another agent?** — the protocols for summoning, delegating, and consulting.
3. **What happens when two agents disagree?** — the conflict escalation path.
4. **How do quality gates work?** — which agent's output gates which other agent's output.
5. **How does code reach `main`?** — branching, commit authorship, merge discipline.
6. **What do the core artefacts look like?** — templates for the acceptance list, demo report, and decisions log.

The roster (`roster.md`) says *who* does *what*. This document says *when* and *how*.

## Git discipline

**No agent commits to `main`. Ever.** This is the single most important process rule in the plugin and it applies to every agent that writes files. The rule is unconditional: no exceptions for "small changes", "just docs", or "Autonomous JFDI mode".

All work happens on branches. How branches reach `main` depends on the project's **code review platform** declared by the ProductOwner in `vision/constraints.md`:

- **Remote-platform flow** (`github | gitlab | bitbucket`) — branches reach `main` via a pull/merge request. The platform CLI (`gh` or `glab`) handles it. RepoSteward pushes the branch, opens the MR/PR, requests review, and merges on approval.
- **Local-only flow** (`none`) — branches reach `main` via a local `git merge --no-ff` performed by RepoSteward after Verifier's demo records *Ready-to-advance: Yes*. No remote push, no platform round-trip. The demo report is the review of record.

In both flows, **no agent commits to `main` directly**. The local-only flow still uses feature branches and still requires Verifier sign-off on a demo before the branch is allowed to merge — it just shortcuts the platform round-trip when no platform exists.

**One agent owns branch lifecycle: `RepoSteward`.** No specialist creates, checks out, merges, or deletes branches. Specialists commit content to whatever branch is currently checked out; RepoSteward sets up the branch at stage start and tears it down at stage close, on instruction from the TeamLead.

**Per-stage branch lifecycle (applies to every stage with its own branch):**

1. TeamLead opens the stage team (`TeamCreate`), spawns core teammates including `repo-steward-<team-surname>`.
2. TeamLead's first task, assigned to RepoSteward: *"Open branch `feature/<stage-slug>` from `main`."*
3. RepoSteward checks the working tree is clean, checks out `main`, creates and checks out the new branch, marks its task `completed` via `TaskUpdate`.
4. TeamLead's next tasks (assigned to the stage's content-producing specialists) run on that branch.
5. When the stage is content-complete, TeamLead's closing task to RepoSteward: *"Close branch `feature/<stage-slug>` — merge to `main`, delete the branch."*
6. RepoSteward reads the code review platform, runs the appropriate local-only (`git merge --no-ff`) or remote-platform (push + PR + merge) sequence, deletes the branch, marks its task `completed`. The merge-commit hash goes in the final `TaskUpdate`'s description (or a brief SendMessage nudge to TeamLead if useful for the lead's status block).
7. TeamLead tears down the team (`TeamDelete`) or proceeds to the next stage.

### Commit authorship — agents identify themselves

**Every agent commit uses `git commit --author="<TeammateName> <slug@jfdi-agents.invalid>"`.** The `--author=` flag sets the commit's **author** field to the agent; the **committer** field stays as the human's configured identity (so GPG signing, push permissions, and platform identity all work unchanged).

Why this matters:

- Without `--author=`, every agent commit shows up in `git log` as if the human wrote it. After session compaction, the TeamLead reads the log, sees "<human> committed X", and concludes the human is actively driving — wonky behaviour follows.
- The git blame record becomes dishonest.
- Post-incident analysis (which agent wrote the broken commit, was it the architect's decision or the backend-dev's implementation?) becomes impossible.

**The canonical form:**

```bash
git commit --author="<TeammateName> <slug@jfdi-agents.invalid>" -m "<subject>

<body>"
```

Where `<slug>` is the teammate name lowercased with no colons or spaces.

**Examples:**

- `backend-dev-skeleton`: `git commit --author="backend-dev-skeleton <backend-dev-skeleton@jfdi-agents.invalid>" -m "feat(backend): add tasks HTTP route"`
- `architect-arch`: `git commit --author="architect-arch <architect-arch@jfdi-agents.invalid>" -m "docs: initial architecture doc and folder map"`

Teammate names are **always** all-lowercase kebab-case. See `${CLAUDE_PLUGIN_ROOT}/docs/team-lead-playbook.md` for the full naming convention.

The `.invalid` TLD is reserved by RFC 2606 for non-deliverable addresses; using it means the author email looks sensible but cannot accidentally be a real address.

The plugin ships `${CLAUDE_PLUGIN_ROOT}/skills/commit-as-agent/SKILL.md` to make this easy — every agent that commits invokes that skill.

### Commit cadence — each stage is one commit boundary; Build layers and Refine passes commit multiple times

Stages have different cadences:

| Stage | Cadence | Who commits |
|---|---|---|
| Intake | 1× at end | ProductOwner |
| Architecture & team design | 1× at end, bundling vision/acceptance.md, docs/architecture.md, docs/decisions.md, and the minted .claude/agents/*.md files | Architect (some files may have PO co-author tags — see below) |
| Build — per layer | N× per layer (one per meaningful milestone inside the layer, minimum one) | The spawned Developer for that layer |
| Build — demo | 1× per Build milestone demo | Verifier |
| Refine pass | N× per pass (one per acceptance item made real, minimum one; developers can work in parallel so their commits interleave) | The spawned Developers for the pass |
| Refine demo | 1× per pass demo | Verifier |

**Co-authoring.** When ProductOwner and Architect jointly write `vision/acceptance.md`, the commit is authored by whichever one is finalising it; the other is added via `Co-authored-by:` in the commit trailer. The canonical merge commit stays RepoSteward's.

**No separate session-note commits.** The plugin does not require session notes. If TeamLead chooses to leave a continuity note for a future session, it goes on the current branch alongside whatever the stage produced.

## The canonical flow — one shell command, one agent team, one session

There is exactly one supported way to run this plugin: the human runs **`./jfdi.sh`** in a project that has been bootstrap-ed for jfdi-agents. The launcher exports `CLAUDE_CONFIG_DIR=$PWD/.claude-state` (per-project Claude home, isolated from `~/.claude`) and execs `claude`; the bootstrap-generated `.claude/settings.json` declares `jfdi-agents:team-lead` as the default agent, so no `--agent` flag is needed. The TeamLead's first action asks the human to authorise creation of a Claude Code agent team, and from that point the TeamLead is the lead of a series of stage-scoped teams. The TeamLead is the only agent that talks to the human directly; every other teammate relays user-facing questions through the TeamLead via `SendMessage`.

`./jfdi.sh` is generated by `/jfdi-agents:bootstrap` and is committed to the repo; it works for anyone who clones the project. There is no equivalent that does not pin `CLAUDE_CONFIG_DIR` — sharing `~/.claude` across multiple JFDI projects is unsupported because parallel teams collide on the shared teams/tasks directories.

The TeamLead itself never authors artefacts. It conducts; the specialists build.

**Checkpointed mode** (default) pauses for explicit human approval at every load-bearing transition.

**Autonomous JFDI mode** runs straight through to the end of the acceptance list without pausing, only stopping on a specialist blocker or a cross-folder technical dispute the Architect cannot resolve with information on hand.

**There is no supported "per-role launch".** Specialists are not designed to run as their own main session, and they do not have access to `AskUserQuestion` — that tool is disallowed in each specialist's frontmatter. The only supported main-session invocation is the TeamLead.

## Session order for a new project

### Step 0 — Install and bootstrap (one time)

```
/plugin marketplace add dr-zog/ai-marketplace
/plugin install jfdi-agents@dr-zog
/reload-plugins
/jfdi-agents:bootstrap
```

Bootstrap lays down the `vision/` and `docs/` directories (empty scaffolds), writes a comprehensive `.claude/settings.json` (default agent, output style, env vars, bypassPermissions, sandbox, marketplace pre-registration, plugin auto-enable), creates `.claude/agents/` ready for the Architect's minted developers and `.claude-state/` for per-project Claude home, and generates `./jfdi.sh` (the launcher that pins `CLAUDE_CONFIG_DIR=$PWD/.claude-state` and execs `claude`).

### Step 1 — Launch the TeamLead

```
/exit
./jfdi.sh     # launcher pins CLAUDE_CONFIG_DIR; settings.json picks the TeamLead as the default agent
```

### Stage 0 — Team creation

TeamLead's first action: `AskUserQuestion` confirming agent-team creation. On confirmation, the Vision-intake team is created.

### Stage 1 — Vision intake (ProductOwner teammate, interactive)

ProductOwner runs the intake interview via the relay pattern. Outputs: all of `vision/` except `acceptance.md`. One commit at stage close; RepoSteward merges.

### Stage 2 — Architecture & team design (Architect + ProductOwner)

Architect reads the Vision, authors `docs/architecture.md` with the four required sections (Layers, Folder map, Developer roster, Technology stack), seeds `docs/decisions.md` with initial pinning calls, and uses the `write-agent` skill to mint one `.claude/agents/<layer>-dev.md` file per developer the project needs. ProductOwner and Architect jointly author `vision/acceptance.md` — the numbered acceptance list in prose.

TeamLead's checkpoint before advancing: verify all minted developer agent files exist in `.claude/agents/` and parse. The `/agents` slash command or a direct `ls` is sufficient.

### Stage 3 — Build (walking skeleton)

For each layer, bottom-up:

1. **Architect picks the layer.** Typically data first, then the layer above it, then the top layer.
2. **TeamLead opens a Build-layer branch.** `feature/skeleton-<layer>`.
3. **TeamLead spawns the layer's Developer.** Teammate name: `<layer>-dev-skeleton`. Teammate task: *"Build the thinnest possible <layer> slice that supports the acceptance list. Stubs for everything not yet needed. Commit incrementally."*
4. **Architect is on the team, available.** Developer messages Architect for any cross-folder decisions.
5. **TeamLead's poll loop notices when the Developer's task transitions to `completed`.** It then creates a Verifier task and (if Verifier isn't yet on the team) spawns it. Verifier runs whatever acceptance items are demonstrable at this slice, writes `docs/demos/<date>-skeleton-<layer>.md`. On Ready-to-advance: Yes, TeamLead closes the branch (RepoSteward merges). On Not-yet, the fix goes back to the Developer on the same branch.
6. **Advance to the next layer.** Repeat.

**When the top layer lands**, one more Verifier pass runs the **full acceptance list** end-to-end. The demo is `docs/demos/<date>-skeleton-complete.md`. That demo's Ready-to-advance: Yes is the gate to Stage 4.

### Stage 4 — Refine (parallel work)

Loop until the acceptance list is fully green or the human says stop:

1. **TeamLead reads the latest demo** and picks the next batch of acceptance items to address.
2. **TeamLead groups items by folder-touched.** Items that touch only one folder can be implemented in parallel with items that touch different folders. Items that touch two folders coordinate via the Architect.
3. **TeamLead opens a Refine branch.** `feature/refine-<N>` where N is a monotonic counter.
4. **TeamLead spawns one Developer per folder being touched in this pass**, simultaneously. Teammate names: `<layer>-dev-refine-<N>`.
5. **Developers work in parallel**, each in their own folder. They commit incrementally. If one needs something from another's folder, they ask Architect via TeamLead relay.
6. **When all developer tasks transition to `completed`**, TeamLead's poll loop creates a Verifier task and spawns Verifier. Verifier runs the full acceptance list, writes `docs/demos/<date>-refine-<N>.md`. On Ready-to-advance: Yes, RepoSteward merges.
7. Increment N and repeat.

## Handoff protocols

### The teammate model

This plugin's design is built on Claude Code [agent teams](https://code.claude.com/docs/en/agent-teams). **Specialists are teammates, not one-shot subagents.** Get this wrong and you hit the harness error *"sub-agent spawning isn't wired into this harness"* — because teammates cannot spawn further teammates; only the lead (TeamLead) can.

**The idiom, in four steps:**

1. **The TeamLead is the lead.** It spawns every specialist as a named teammate and is the only agent that can add new teammates to the running team.

2. **Teammates are named.** When the TeamLead spawns a specialist, it gives that teammate an explicit name, all-lowercase kebab-case. For work-unit-scoped specialists, suffix accordingly: `backend-dev-skeleton`, `frontend-dev-refine-3`.

3. **Consultations are `SendMessage` tool calls to a named teammate.** When agent A needs an answer from agent B, A invokes `SendMessage` with `to: "<B's teammate name>"`. Writing the message as prose in your output does nothing; the harness needs the tool invocation. A does *not* attempt to spawn B itself.

4. **If the needed teammate isn't on the team yet**, A `SendMessage`s the **lead** (addressed as `team-lead`; see next paragraph) with a request of the shape *"please add Architect to the team so I can consult on X, or relay the question to Architect on my behalf."*

**The lead's addressable name is `team-lead`, not `TeamLead`.** `TeamCreate` returns `lead_agent_id: "team-lead@<team-name>"`, and `team-lead` is the name `SendMessage` expects. "TeamLead" is the lead's **role**, not its addressable identity. Getting this wrong is silent: `SendMessage({to: "TeamLead", ...})` returns `success: true`, writes to a dead-letter inbox for a recipient that doesn't exist, and the lead never sees the message body.

**Every agent prompt in this plugin that mentions `SendMessage` to the lead must use `to: "team-lead"` explicitly.** Role-label shorthand silently fails.

### The question brief must be narrow and self-contained

The receiving teammate does not see your conversation history. Write a brief that stands alone:

Bad:
> "Hey Architect, can you look at this?"

Good:
> "Architect: I'm in backend-dev implementing acceptance #5 (a user can delete a task and it disappears from their list). The frontend-dev's current TaskList component polls GET /tasks every 3s. If I implement DELETE /tasks/:id to return 204 No Content, the frontend should see the delete within 3s. Is that acceptable, or do you want pub/sub? Please reply via SendMessage with your answer."

### The two channels — Tasks for state, SendMessage for nudges

**Tasks are the work-and-state channel.** Every action a teammate is expected to take lives in a Task with formal `owner`, `blockedBy`, `status`, and description. State transitions happen via `TaskUpdate`:

- A teammate **starts** work by calling `TaskUpdate(status: "in_progress")` on the task it owns.
- A teammate **finishes** work by calling `TaskUpdate(status: "completed")`. The artefact on disk (committed to the branch) is the payload; the status transition is the signal. There is no "DONE:" text marker to send.
- A teammate that hits a dependency it cannot resolve raises it as new task state — either by adding a `blockedBy` entry to its own task, or by creating a follow-up task it owns and pointing the original at it. The TeamLead's periodic-poll loop (see `${CLAUDE_PLUGIN_ROOT}/docs/team-lead-playbook.md` § "The periodic-poll loop") notices state changes and reacts.

**SendMessage is the nudge-and-out-of-band-Q&A channel.** Legitimate uses:

- A teammate needs a clarification before continuing — *"Architect: I'm on backend's task. The contract with frontend on X is unclear. Please reply via SendMessage."*
- A teammate needs to relay a question to the human — *"team-lead: ProductOwner has a wording question for the human; brief follows."*
- The TeamLead checks in on an agent whose task has been `in_progress` with no state change for several minutes — *"team-lead → backend-dev: status check, what's blocking you?"*
- Architect SendMessages disputing parties with a ruling after writing it to `docs/decisions.md`.

**SendMessage is NOT used for state transitions.** Don't send "DONE:" or "BLOCKED:" messages. Don't summarise completion in a SendMessage and assume the lead noticed. The task channel is the only authoritative state. The TeamLead's loop reads task state; if it isn't in the task, the lead does not act on it.

**Why this split.** An earlier eval run had RepoSteward jump a `blockedBy` gate because a SendMessage body suggested it could proceed. The competing-channels architecture is what made that possible. Under one ground-truth channel (Tasks) for state, the failure mode disappears at source.

### SendMessage is a tool invocation, not prose (load-bearing)

**Writing a reply as text in your conversation output does NOT send it to the requester.** The reply only reaches the other teammate if you invoke the `SendMessage` tool with `to: "<teammate-name>"`. This sounds obvious and has still caused at least one deadlock in an eval run.

**The discipline for every teammate:**

1. **After drafting your reply content**, the last thing you do is invoke the `SendMessage` tool. Confirm the tool-use block appears in your output. If it does not, the message was not sent.
2. **When you send a question that expects a reply**, include the literal phrase **"Please reply via SendMessage with your answer"** in the question brief. Do not assume the receiver understands a reply is expected — make the tool-invocation requirement explicit.
3. **If you think you've replied but see no `SendMessage` tool-use block in your output**, assume you haven't and send again.

The `${CLAUDE_PLUGIN_ROOT}/scripts/stall-detector.sh` exists in part to catch the failure mode when this discipline slips — see `${CLAUDE_PLUGIN_ROOT}/docs/team-lead-playbook.md`. The detector is the safety net; the discipline is the primary fix.

### Wait semantics

Once a teammate is spawned, it runs in parallel in its own session. Message delivery is automatic. Teammate idle between turns is normal, not stall. Do not ping an idle teammate.

### Parallel consultations on the running team

Once a teammate exists, anyone on the team can `SendMessage` to it. Typical parallel cases in Refine: two developers simultaneously asking Architect different questions; Verifier querying ProductOwner on an acceptance wording while Architect is reviewing a dispute.

Parallelism is cheap, but *naming collisions* are expensive. The TeamLead — as the only agent that can spawn — is responsible for unique names.

## Conflict escalation

The plugin has **no Mediator role**. Disputes resolve at one of two places depending on type:

### Technical disputes — Architect decides

Any disagreement about *how* to build something. Examples:
- Backend-dev wants REST; frontend-dev wants GraphQL.
- Data-dev wants to denormalise; backend-dev wants the normalised version.
- A developer thinks the acceptance wording requires behaviour different from what the Architect envisaged.

**Flow.**
1. The parties try to resolve in one exchange via SendMessage.
2. If they don't converge, each party writes a two-paragraph position statement (what they want, why — referenced to an owned artefact or a declared prior).
3. Either party SendMessages Architect with both statements verbatim. Architect rules. The ruling is appended to `docs/decisions.md` as a one-liner and sent back to both parties via SendMessage. Both parties are bound.

**Why this works here.** The Architect is the de facto technical authority on every project this plugin runs, already owns the decisions log, and is present through every stage. Spawning a separate Mediator is extra machinery for no benefit.

### Operational disputes — TeamLead decides

Any disagreement about *who does what, when*. Examples:
- Two developers both need Architect's time at the same moment.
- A developer claims an acceptance item is for their folder; another claims it's for theirs.
- A developer says they're blocked on Verifier; Verifier says they're done.

**Flow.**
1. Either party SendMessages TeamLead with a short *"situation, question"* brief.
2. TeamLead surveys state, decides, replies via SendMessage. The decision is operational (scheduling, task reassignment) and goes in the task list, not the decisions log.

### Escalate to human

If a technical dispute cannot be resolved by the Architect (typically: the decision requires product-level trade-offs the Architect isn't licensed to make), Architect writes a short brief and SendMessages TeamLead, which presents it to the human via `AskUserQuestion`.

## Quality gates

| Upstream output | Downstream output it gates |
|---|---|
| `vision/overview.md` | anything in `docs/` |
| `vision/constraints.md`'s architectural components list | `docs/architecture.md`'s Layers section |
| `vision/constraints.md`'s code review platform line | RepoSteward's merge flow |
| `docs/architecture.md` with all four required sections | `.claude/agents/<layer>-dev.md` files (developers are minted *after* architecture is agreed) |
| All `.claude/agents/<layer>-dev.md` files exist and parse | Stage 3 (Build) starts |
| `vision/acceptance.md` exists | Stage 3 Verifier passes |
| Walking-skeleton complete with Verifier's Ready-to-advance: Yes | Stage 4 (Refine) — parallel work is licensed |
| Full acceptance list green on Verifier | Project may be declared complete |

Attempting to produce a downstream output without the upstream prerequisite is a process violation and must be flagged to the human.

## The sequential-skeleton rule (load-bearing)

**Parallel work is forbidden before the walking skeleton exists.** This is the single most common failure mode of autonomous agent teams in this plugin's lineage — developers racing ahead on parallel layers before the integration surfaces have been settled, resulting in three "done" layers that don't actually compose.

- **TeamLead** enforces this when scheduling. In Stage 3, at most one developer is spawned at a time. Violations are surfaced to the human.
- **Architect** enforces this by refusing to sign off on a Build-phase demo until the full skeleton runs end-to-end.
- **Verifier** enforces this by marking a Build-complete demo as Not-yet if the skeleton doesn't exercise every acceptance item at least trivially.

## The folder-ownership rule (load-bearing)

**Every developer owns exactly one folder, and edits only that folder.** The Architect writes this into every minted developer body. If a developer thinks it needs to edit outside its folder, it raises a `blockedBy` on its task (or creates a follow-up task and points the original at it) and asks Architect for the ruling via SendMessage.

- **Verifier** audits every demo for cross-folder writes. `git diff --stat <branch>..main` against the folder map catches them.
- **TeamLead** routes Verifier's CRITICAL cross-folder findings back to Architect (who either revises the folder map and re-mints, or tells the developer to undo).

The only exception: **the shared layer**, if the Architect declared one. Its developer owns `shared/` (or whatever path); other developers consume from it but do not modify it.

## The agent-teams tool surface — call the tools

Every reference to "the team", "spawning a teammate", "messaging a teammate", "the task list", or "cleaning up the team" corresponds to a **concrete Claude Code tool call**:

| Plugin concept | Claude Code tool | Who calls it | Carries… |
|---|---|---|---|
| Create the team | `TeamCreate` | TeamLead (lead), once per stage | — |
| Disband the team | `TeamDelete` | TeamLead (lead), at stage end | — |
| Add a teammate | `Agent` (in an active-team session) | TeamLead (lead) | — |
| Send a nudge or out-of-band Q | `SendMessage` | any agent, by teammate name | **prose only — never state transitions** |
| Create a task | `TaskCreate` | TeamLead (lead); teammates should not | the work brief |
| Inspect task state | `TaskList`, `TaskGet` | any agent | the source of truth |
| Update task state | `TaskUpdate` | TeamLead for all tasks; teammates for their own | **every state transition** — `in_progress`, `completed`, new `blockedBy` |

All experimental team tools are gated on `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Bootstrap sets this in `.claude/settings.json`; the TeamLead verifies it at session start.

**Three harness behaviours we align with rather than fight:**

1. **The canonical completion signal is `TaskUpdate(status: "completed")`, not a text marker.** The task-list status transition is the state. SendMessage is for nudges and out-of-band Q&A. See "The two channels" above.
2. **Teammate idle between turns is normal, not stall.**
3. **Messages arrive automatically as conversation turns.** No polling, no inbox-scan loops — but the TeamLead does run a periodic `TaskList` loop to react to teammate-driven state changes; see `${CLAUDE_PLUGIN_ROOT}/docs/team-lead-playbook.md` § "The periodic-poll loop".

## Three invocation patterns: lead, teammate, solo

The plugin supports three distinct agent invocation patterns. The first two are the dominant flow this document covers. The third is a deliberate breakout documented separately.

| Pattern | Entry point | `AskUserQuestion` | Reference |
|---|---|---|---|
| **Lead** | `./jfdi.sh` in a bootstrap-ed project (the launcher exports `CLAUDE_CONFIG_DIR` and execs `claude`; settings.json picks `jfdi-agents:team-lead`) | Yes — direct human channel | `${CLAUDE_PLUGIN_ROOT}/docs/team-lead-playbook.md` |
| **Teammate** | Spawned by the lead via `Agent` into the running team | No — `disallowed-tools: AskUserQuestion`, relays through the lead | This document, throughout |
| **Solo** | `claude --agent <name>` (main session, human-paired, outside the team flow) | Yes | `${CLAUDE_PLUGIN_ROOT}/docs/solo-agents.md` |

The TeamLead is the only supported main-session launch within the team flow. Per-role main-session launches of teammate agents (e.g. `claude --agent jfdi-agents:product-owner`) are not supported — teammates only function inside the lead's team. Solo agents launch as their own main session by design but are NOT teammates and NOT part of the team flow; see `solo-agents.md` for when solo mode is licensed, the human-gating protocol that authorises minting one, and how solo work integrates with subsequent team sessions.

## Timescale never licenses shortcuts

This is a standing rule binding on every agent.

**The workflow is non-negotiable regardless of how fast or slow the project is expected to ship.** The sequential-skeleton discipline, the one-folder-per-developer rule, the Verifier sign-off between stages — all of these run regardless.

**This rule exists because urgency language in a Vision or kickoff prompt primes LLMs to take shortcuts.** Training data is full of time-pressured software projects where corners got cut; an LLM reading "this week" or "urgent" pattern-matches onto that.

**What is still allowed to influence the workflow:**
- **Hard external deadlines** — regulatory dates, contract commitments. Captured in `vision/constraints.md`. These might influence which acceptance items land first, but they never license skipping a stage.

**What is not allowed:**
- "In the interest of time, parallelise before the skeleton" — no. Skeleton first.
- "This project is small, so skip Verifier" — no. Verifier runs between every stage.
- "Autonomous JFDI means go fast" — no. Autonomous JFDI means *no human checkpoint*, not *fewer checks*.

If any agent finds itself rationalising a skip with a timescale argument, stop. Complete the step.

## Session hygiene

- **Interruption safety.** Never leave a partial artefact on disk. Write-then-rename if needed.
- **No silent edits to owned artefacts of another agent.** If Developer needs the architecture doc updated, they ask Architect via the TeamLead relay.
- **No session notes required.** TeamLead may leave a short note at `docs/sessions/<date>.md` if it helps a future session pick up, but this is optional and discouraged unless genuinely useful.

## Human overrides

A human operator can override any of the above at any time by saying so explicitly. Agents respect the override and (if the override is non-trivial and likely to recur) surface it to the Architect for inclusion in `docs/decisions.md`.

---

# Artefact templates

These templates are referenced from the agent prompts. Use them as the starting point for each artefact; add sections if the project demands, but do not omit sections without noting why.

## Acceptance list template

Used by ProductOwner + Architect at `vision/acceptance.md`:

```markdown
# Acceptance list

Every item in this list is an observable behaviour from the perspective of
an end-user or operator persona (see `personas.md`). Items are numbered;
numbers never change once assigned. New items get the next number with a
date in parentheses.

1. A user can sign up with email and password and is landed on the dashboard.
2. A user can sign in, and is redirected to the dashboard.
3. A signed-in user can add a task to their list by typing and pressing Enter.
4. A signed-in user can mark a task as complete, and it visibly moves to the completed section.
5. Tasks persist across app restarts.
6. An operator can run `npm start` and see the system come up on localhost:3000.
7. **(added 2026-04-29)** A user can rename a task by clicking the title and editing.
```

## Demo report template

Used by Verifier at `docs/demos/<YYYY-MM-DD>-<slug>.md`:

```markdown
# Demo: <slug>

**Date:** <ISO-8601 timestamp>
**Verifier:** <teammate name>
**Stage:** Build — <layer> | Build — complete | Refine pass <N>
**Ready-to-advance:** Yes | Not yet

## Environment

- OS: ...
- Runtime versions: ...
- Command(s) run to start the system (verbatim):
  ```
  <exact commands>
  ```

## Acceptance-list status

| # | Acceptance item (abbreviated) | Status | Notes |
|---|---|---|---|
| 1 | sign up and land on dashboard | PASS | Tested via curl; UI not yet present. |
| 2 | sign in | PASS | |
| 3 | add a task | NOT YET | Frontend layer not yet built — deferred to next layer. |
| 4 | mark task complete | NOT YET | |
| 5 | tasks persist | PASS | Verified via restart of the backend process. |
| 6 | operator runs `npm start` | PASS | |

**Totals:** <P> pass / <F> fail / <N> not-yet / <T> total.

## Three-axis audit

### Completeness
<Which acceptance items are covered by the current work? Cite what was exercised and how.>

### Correctness
<Does observed behaviour match the acceptance wording? Cite specific checks.>

### Coherence
<Does the code stay within folder ownership? `git diff --stat` shows edits touched only: <folders>. Any cross-folder writes are a CRITICAL finding below.>

## Findings

### CRITICAL
- <file:line> <description> — <which agent should fix (backend-dev / architect / etc.)>

### WARNING
- <...>

### SUGGESTION
- <...>

## Recommendation

<One paragraph. If "Not yet", name the CRITICAL findings that must be addressed.>
```

## Decisions log format

Used by Architect at `docs/decisions.md`:

```markdown
# Decisions

Append-only one-line record of non-obvious technical calls. A decision is
logged when a future reader would ask "why did we pick this?" and have no
answer. Obvious defaults (use npm for JavaScript, use pytest for Python) are
not logged.

- **2026-04-22** — Pinned Postgres 16. SQLite considered; rejected because acceptance #7 requires concurrent writes.
- **2026-04-22** — Backend language: TypeScript + Fastify. Chosen for familiarity and fast startup.
- **2026-04-22** — Frontend language: React + Vite. Default for the stack.
- **2026-04-23** — backend-dev vs frontend-dev on delete-task semantics: REST DELETE /tasks/:id returning 204; frontend polls. Architect ruled for simplicity; revisit if latency becomes a complaint.
```

No frontmatter. No sub-sections. The format is deliberately flat — it's a log, not a reference work.
