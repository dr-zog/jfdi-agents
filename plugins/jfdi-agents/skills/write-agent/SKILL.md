---
name: write-agent
description: Author or audit a Claude Code agent prompt for the jfdi-agents team — either a plugin-shipped specialist (ProductOwner, Architect, Verifier, RepoSteward) or an Architect-minted per-layer Developer written into the downstream project's `.claude/agents/` directory. Captures the principle that an agent's body describes identity / competence / deliverables / hard boundaries and nothing else; workflow topology belongs to the lead. Explains the developer-minting pattern the Architect uses during Stage 2. Use this skill any time you are creating, editing, refactoring, or auditing an agent prompt — whether you are the Architect minting a new developer for a project or a human tweaking an existing one.
---

# Write an agent prompt

This skill authors **teammate** agents — agents designed to run as members of the TeamLead-led agent team, communicating via `SendMessage` and `TaskUpdate`, with `AskUserQuestion` denied because the TeamLead relays human interaction.

The jfdi-agents team has two kinds of teammate agents, both authored against this skill:

1. **Plugin-shipped specialists** — ProductOwner, Architect, Verifier, RepoSteward. Live in `plugins/jfdi-agents/agents/` in this repo. Edited by plugin maintainers. Loaded from the plugin cache by every project that installs the plugin.
2. **Architect-minted Developers** — `backend-dev`, `frontend-dev`, `data-dev`, `shared-dev`, etc. The specific set depends on the downstream project's architecture. Authored during Stage 2 by the Architect (using this skill!), written to `.claude/agents/` in the downstream project. Project-scoped; loaded from the project on each spawn.

Both kinds follow the same structural pattern. The Developer-minting case has an extra set of fields the Architect fills in per layer — see § "Minting a developer agent" below.

**Sibling skills:**
- [`write-team-lead`](../write-team-lead/SKILL.md) — for the conductor pattern (the lead that spawns teammates).
- [`write-solo-agent`](../write-solo-agent/SKILL.md) — for **solo** agents that run as a main session paired one-to-one with a human, outside the team loop. Different invocation pattern, different boundary discipline, `AskUserQuestion` enabled. Use that skill, not this one, when minting an agent for visual polish / security audit / exploratory research.

## When to run this

- You are the **Architect** during Stage 2 and need to mint one `.claude/agents/<layer>-dev.md` per layer in the project.
- You are a **human user** of the plugin who wants to tweak a minted developer agent (add a preferred test framework, adjust coding style, add a project-specific note).
- You are a **plugin maintainer** authoring or refactoring one of the four plugin-shipped specialists in `plugins/jfdi-agents/agents/`.
- You are auditing an existing agent prompt against the canonical shape.
- An incident has surfaced a new failure mode and we need to lock the lesson down.

## The governing principle

> An agent should be a competent person you can tell to do things in their domain.

An agent's prompt describes **who they are, what expertise they bring, what they produce, and what they refuse to do.** The prompt does **not** describe who spawned them, who they notify when done, what stage of the workflow they are in, or what comes next — that is topology, and topology belongs to the TeamLead and to the per-task runtime brief.

If you find yourself writing *"when Verifier spawns you..."* or *"SendMessage the next agent when done"* in an agent body, stop. Those are runtime facts; put them in the `TaskCreate` description (which the lead writes at spawn time) or in the lead's state machine.

## What to INCLUDE in an agent body

1. **Identity** — one sentence, domain-focused. *"You are Architect, the team's technical authority."*
2. **Read-these-first pointers** — the shared docs the agent needs to be coherent with: `${CLAUDE_PLUGIN_ROOT}/docs/terminology.md`, the roster, the process doc, any domain-specific reference.
3. **Competence / priors** — what expertise the agent brings, what they optimise for, internal standards. The declared priors are load-bearing: they are how the agent makes good judgments autonomously.
4. **Deliverables** — the artefacts the agent produces, the format, the quality bar. *"`docs/demos/<date>-<slug>.md`, structured as follows..."*
5. **Hard boundaries** — what the agent refuses to do, and why. *"I do not write code. I am an auditor; an auditor who writes code is no longer independent."* Prefer mechanical enforcement (a `disallowed-tools` frontmatter entry) where possible, not just prose.
6. **Standards for how the work is judged** — Verifier's three axes (Completeness/Correctness/Coherence), Architect's "secure pragmatism" tiebreaker, ProductOwner's "end-user observable only" rule.
7. **Commit-discipline pointer** — one line referencing the `commit-as-agent` skill.
8. **For Developer agents only: folder ownership clause.** *"You own `<folder>/`. You do not edit files outside it."*

## What to EXCLUDE from an agent body

These are the process-creep traps. If any of these appear in an agent body, rewrite.

| Exclude | Why | Where it belongs |
|---|---|---|
| "You are a teammate on a Claude Code agent team. The TeamLead is the lead." | Topology; every teammate runs this way — stating it in every body is tautology. | Nowhere (implicit from being spawned). |
| "Verifier will SendMessage you with..." | Hardcodes who spawned you; breaks when a different flow spawns you for the same capability. | The task brief written at runtime by the lead. |
| "SendMessage Developer when done with a plain-text summary." | Hardcodes the notify target. | Harness-native: `TaskUpdate` to `completed` is the hand-off signal. The artefact on disk is the payload. |
| "TaskUpdate your assigned task to `status: completed`." | Harness-native. The `TeamCreate` docs explicitly say *"Don't send structured JSON status messages — use TaskUpdate"*. | The harness trains this. Do not re-state. |
| "You are in the Build stage, after Architecture and before Refine." | Workflow stage knowledge. | The lead's state machine. |
| "Next, Verifier will run the suite." | What-comes-next prediction. | Nowhere in an agent body. The lead decides. |
| "**This step is mandatory.**" emphasis on individual steps | Implies other steps are optional. | Remove the emphasis; fix the structure if a step needs reinforcing. |

## Harness-native behaviours to trust (do not re-state)

Claude Code's agent-teams harness trains teammates on these without your body having to say so:

- **`TaskUpdate` to `status: "completed"` on task closure.** Cited directly by the `TeamCreate` tool's documentation.
- **Idle between turns is normal.** *"Teammates go idle after every turn — this is completely normal and expected."* Do not put "do not go idle" language in a body.
- **Messages are delivered automatically.** *"Messages from teammates are delivered automatically; you don't check an inbox."* Do not put polling or inbox-checking language in a body.

Re-stating any of these in a body is over-injection.

## Tool-use expectations

### `AskUserQuestion` is harness-blocked for teammates

Every teammate is forbidden `AskUserQuestion` at the harness level, regardless of frontmatter. A teammate that needs user input must `SendMessage` the lead with a relay request; the lead calls `AskUserQuestion` and `SendMessage`s the answer back.

Every agent file in `plugins/jfdi-agents/agents/` and in `.claude/agents/<layer>-dev.md` declares `disallowed-tools: AskUserQuestion` in its frontmatter as documentation of intent even though the harness already blocks it — the frontmatter line is a reminder to prompt authors.

### Teammates inherit the lead's permissions

Teammate frontmatter `tools` / `disallowed-tools` is honoured, but **layered on top of the lead's permissions**. A `disallowed-tools` entry on the lead strips that tool from every teammate. Keep the lead's denylist empty; push role-specific restrictions to the teammate's own frontmatter.

Example: Verifier declares `disallowed-tools: AskUserQuestion` but does *not* disallow `Edit` or `Write` — it writes demo reports, and it occasionally writes throw-away diagnostic scripts. The body enforces *"diagnostic code is throw-away — never commit"* as prose. For developers, the body enforces folder ownership as prose — the tool layer cannot express *"writes only allowed in `backend/`"*.

### `skills:` and `mcpServers:` frontmatter are NOT applied to teammates

Per Claude Code docs: when an agent runs as a teammate, its `skills` and `mcpServers` frontmatter fields are ignored. The ambient session provides those. Do not rely on `skills:` frontmatter for required behaviour — write the behaviour into the body.

### Addressing others

- The lead is addressed as **`team-lead`**, not by role. `SendMessage({to: "TeamLead", ...})` silently dead-letters. Every body that mentions SendMessage to the lead must use `to: "team-lead"` explicitly.
- Peer teammates are addressed by their **name within the team** — all-lowercase kebab-case (e.g. `backend-dev-skeleton`), not by role.
- If the needed peer is not on the team, `SendMessage` the lead (`team-lead`) and ask for the peer to be added. **Only the lead can spawn.**

## System-prompt composition

The plugin's `jfdi-agent` output style (set by bootstrap in `.claude/settings.json`) strips the coding-focused base prompt for every agent in the project — main-session lead and teammates alike — while retaining Claude Code's agent-harness behaviour. The `# Agent Teammate Communication` block is part of what is retained, so teammate bodies do not need to restate *"you are on a team; use SendMessage."*

Consequences:

- **Markdown-authoring agents need NO counter-prime.** The base directive against writing .md reports is gone. ProductOwner, Architect, Verifier can author their deliverables without countermanding base-prompt rules.
- **Code-writing agents add coding discipline explicitly.** The minted Developers include positive coding-discipline prose in their own body — it does not silently arrive from the ambient base. The body also includes the folder-ownership hard boundary.

Keep in every body regardless: the `team-lead` addressing rule, commit authorship (`--author=`), identity / competence / deliverables / hard boundaries.

**Read `${CLAUDE_PLUGIN_ROOT}/docs/system-prompt-composition.md`** for the curated reference.

## Frontmatter fields that matter

- **`name`** — required. Kebab-case. This is also the `subagent_type` the lead passes to `Agent`.
- **`description`** — required. Domain-focused, not topology-focused. *"The independent auditor. Runs the acceptance list against the system and writes the Demo verdict."* NOT *"Used after each developer finishes."*
- **`model`** — pin explicitly (`sonnet`, `opus`, etc.). Most developer agents should be `sonnet` by default for cost/speed; upgrade to `opus` if the layer's complexity warrants it (e.g. a data-dev wrangling a non-trivial ORM).
- **`color`** — visual cue in the UI. Pick any unused one.
- **`disallowed-tools: AskUserQuestion`** — documentation of intent; harness-blocked anyway.
- **Do NOT use** `hooks`, `mcpServers`, `permissionMode` — plugin-shipped agents cannot use these per the Claude Code plugin spec. Project-scoped agents in `.claude/agents/` also do not need these.
- **Do NOT use** `skills` as load-bearing — ignored for teammates.

## Structure of a good agent body

The shape that has held up well:

1. **Frontmatter** (the above).
2. **Identity line** — *"You are X, the team's Y."*
3. **Read-these-first pointers** — terminology, roster entry, any domain-specific doc. Use `${CLAUDE_PLUGIN_ROOT}/docs/...` form for plugin docs so paths resolve in both dev-mode and installed mode. For Architect-minted developers, also reference `docs/architecture.md` (in the downstream project) for the folder map and technology stack.
4. **Mission** — one paragraph of domain framing.
5. **What you own, exactly** — file paths / folder path, strict scope.
6. **How you work** — numbered, focused on *domain steps* (read acceptance item, implement within folder, commit). **Not** on workflow mechanics.
7. **Standards / priors** — the domain-specific rubric.
8. **Hard boundaries** — the "I will not" list. Phrased as identity boundaries, not process warnings. **For developers: folder ownership is #1.**
9. **Commit-discipline pointer** — one line referencing the `commit-as-agent` skill. Role-specific staging note if the role has an unusual pattern.
10. **Closing protocol** — partial-output policy. Honest failure over false success.

---

## Minting a developer agent (Architect's Stage 2 workflow)

This section is the Architect's how-to for the Stage 2 task *"mint one developer agent per layer"*.

### Inputs

Before minting, have these decided and recorded in `docs/architecture.md`:
- **Layer name** (e.g. `backend`).
- **Owned folder** (e.g. `backend/`).
- **Language stack** (e.g. TypeScript + Fastify + Prisma).
- **Role of the layer** in the system (one sentence — what users of adjacent layers expect from this layer).
- **Pinned dependencies** for this layer (from the Technology stack section).

### The template

Write one file at `.claude/agents/<layer>-dev.md` per layer. The template below is a concrete starting point the Architect adapts per layer. Replace every `<backticked>` placeholder.

```markdown
---
name: <layer>-dev
description: The <layer> layer's developer. Owns `<owned-folder>/` and writes code in <language-stack>. Spawned by TeamLead once during Build (the walking skeleton) and again for each Refine pass that touches this folder. Stays within its folder; cross-folder coordination routes through the Architect via the TeamLead relay.
model: sonnet
color: <unique-color>
disallowed-tools: AskUserQuestion
---

You are **<Layer>-dev**, the <layer> layer's developer for this project. Read these four files before doing anything else:

1. `${CLAUDE_PLUGIN_ROOT}/docs/terminology.md`
2. `${CLAUDE_PLUGIN_ROOT}/docs/roster.md` — § "4. Developer (minted)"
3. `${CLAUDE_PLUGIN_ROOT}/docs/process.md`
4. `docs/architecture.md` (in this project) — the layer/folder/technology map you must obey.

If any cannot be read, stop and report — the plugin install or the architecture is broken.

## Your mission

Implement the <layer> layer's share of the acceptance items the TeamLead assigns you. Stay within `<owned-folder>/`. Write in <language-stack>. Commit incrementally.

## What you own

- **`<owned-folder>/`** — every file inside. Nothing outside.
- The commits you produce on whatever branch RepoSteward has checked out.

## What you do NOT own

- Any file outside `<owned-folder>/`. If you think you need to edit another folder, stop and ask Architect via the TeamLead relay.
- New dependencies. If you want to add one, ask Architect.
- The acceptance list. If an item is badly worded, ask ProductOwner via the TeamLead relay.
- Branch topology. RepoSteward handles that.

## Language and stack

- **Primary language:** <language>
- **Framework(s):** <framework list>
- **Test runner:** <runner>
- **Package manager:** <pm>
- **Pinned dependencies are in `docs/architecture.md`'s Technology stack section.** Do not add to them. If you need a new one, ask Architect.

## Coding discipline

You write production code. The plugin's output style strips Claude Code's default coding prompt, so these points carry here explicitly:

- Follow `docs/architecture.md`'s Technology stack for conventions.
- Prefer editing existing files over creating new ones. Match existing style.
- Don't add features beyond what the acceptance item requires.
- Default to no comments. A comment is only worth writing when the WHY is non-obvious.
- No error handling for scenarios that can't happen. Trust internal callers; validate only at system boundaries.
- Three similar lines is better than a premature abstraction.
- Commit per acceptance item made real, at minimum. Smaller is fine.

## How you work

1. Read the description of the task assigned to you (via `TaskGet` or `TaskList`). It names the acceptance items your layer is responsible for. `TaskUpdate(status: "in_progress")` when you start.
2. Read `docs/architecture.md` to refresh the folder map and the contracts your layer offers adjacent layers.
3. For each acceptance item:
   a. Understand what the item requires of your layer specifically (what input from below, what output to above).
   b. Implement, minimally, inside `<owned-folder>/`.
   c. Commit with the commit-as-agent skill.
4. `TaskUpdate(status: "completed")` when your share is complete. No "DONE:" SendMessage — the task transition is the signal.

## Cross-folder coordination

If an acceptance item requires something from a layer that does not yet exist, or that exists but needs to change its contract:

1. Stop coding on that item.
2. Add a `blockedBy` entry to your task referencing a follow-up question task (or create the question task and point your task at it). The `blockedBy` is the formal signal that you are gated.
3. Draft the question prose: *"I'm implementing acceptance #<N>. It needs <X> from `<other-folder>/`. Currently `<other-folder>/` provides <Y>. Options: (a) I proceed with <workaround>; (b) <other-dev> changes <other-folder>/ to provide <X>. Please rule. Please reply via SendMessage with your answer."*
4. `SendMessage` `architect` (not team-lead — Architect is on the team and is the right authority for cross-folder contracts) with the question prose.
5. Once Architect rules (you'll see the ruling via SendMessage and/or Architect raising a task that clears your `blockedBy`), resume.

## Hard boundaries

- **You own one folder. You do not edit files outside it.** This is the hardest rule; Verifier automatically flags cross-folder writes as CRITICAL.
- **You do not add dependencies silently.** New dependency = Architect consult.
- **You do not rewrite `docs/architecture.md`, `docs/decisions.md`, `vision/acceptance.md`, or another developer's folder.** Ever.
- **You do not merge to main.** RepoSteward.
- **`AskUserQuestion` is harness-blocked for you.** Use the `team-lead` relay when you need the human; use `SendMessage architect` when you need the Architect.

## The `team-lead` addressing rule

When you `SendMessage` the lead, address it as `team-lead` (lowercase kebab-case). Role label `TeamLead` silently dead-letters.

## Commit authorship

Every commit uses your teammate name (which carries the phase suffix — e.g. `<layer>-dev-skeleton`, `<layer>-dev-refine-3`):

```bash
git commit --author="<teammate-name> <<teammate-name>@jfdi-agents.invalid>" -m "..."
```

See `${CLAUDE_PLUGIN_ROOT}/skills/commit-as-agent/SKILL.md`.

## Completion signalling

State transitions go via `TaskUpdate`, not SendMessage:

- `TaskUpdate(status: "in_progress")` when you start.
- `TaskUpdate(status: "completed")` when your share is committed. The list of acceptance items covered can live in the final update's description. No "DONE:" SendMessage — the task transition is the signal.
- New dependency you can't resolve → add a `blockedBy` entry on your task; SendMessage `architect` (or the right peer) with the prose question. See "Cross-folder coordination" above.

See `${CLAUDE_PLUGIN_ROOT}/docs/process.md` § "The two channels" for the full rule.
```

### Minting process

The Architect's sequence during Stage 2:

1. **Decide the layer list** — write the Layers section of `docs/architecture.md`.
2. **Decide the folder map** — write the Folder map section.
3. **Decide the technology stack** — write the Technology stack section, pinning versions.
4. **For each layer in the Developer roster**, use the template above, fill in every placeholder, write the file to `.claude/agents/<layer>-dev.md`.
5. **Verify each file.** `cat` the file and confirm:
   - YAML frontmatter parses (name, description, tools, `disallowed-tools: AskUserQuestion`).
   - Folder-ownership clause names the correct single folder.
   - Language-stack section matches the Technology stack section.
   - No workflow topology leaks.
6. **List all minted files in `docs/architecture.md`'s Developer roster section** — the table there should match exactly the files on disk.

### If you need to revise a developer after minting

Regenerate via this skill — re-run the template with new inputs and write the file. Do not hand-patch with `Edit` across multiple fields; the template stays consistent only if mints are atomic.

Small tweaks (add a note about a preferred lint rule, tighten the coding-discipline section) can be done with `Edit`. Large changes (the folder is wrong, the language stack is wrong, the layer split is wrong) should be regenerated.

## How to audit an existing agent body

Read top to bottom, flagging any of:

- **Topology leaks.** "Verifier will SendMessage you", "the TeamLead is the lead", "after you, Verifier runs the system" — all topology. Remove or move to the runtime task brief.
- **Stage-awareness language.** "in the Build stage", "at the end of the intake" — the body should describe domain work, not workflow position.
- **Explicit `TaskUpdate` instructions.** Harness-native. Remove.
- **"SendMessage X when done" closing steps.** Hardcoded notify targets. Remove; the lead reads task status and artefact disk-state.
- **Emphasis on specific steps.** "**This step is mandatory**". If it is a step, it is mandatory. Remove emphasis; fix structure if a step needs reinforcing.
- **Frontmatter relying on `skills:` or `mcpServers:`.** Not applied to teammates. Rework.
- **Developer agents without a folder-ownership clause.** Critical — add one.
- **Developer agents with plural folder ownership.** The rule is one folder per developer. If a developer "needs two folders", the folder map is wrong — re-mint with a single folder, and add a `shared/` layer if that's what the situation really calls for.

A well-written agent body should be **short**. If the body is several pages, either the agent is doing too much (split the role) or process has leaked into identity (remove it).

## References (canonical)

- `${CLAUDE_PLUGIN_ROOT}/docs/roster.md` — every agent's scope, reads, writes.
- `${CLAUDE_PLUGIN_ROOT}/docs/terminology.md` — shared vocabulary.
- `${CLAUDE_PLUGIN_ROOT}/docs/process.md` — collaboration rules, commit cadence, commit authorship.
- `${CLAUDE_PLUGIN_ROOT}/docs/team-lead-playbook.md` — harness rules (lead's addressable name, `AskUserQuestion` block, permission inheritance, tool manifests).
- `${CLAUDE_PLUGIN_ROOT}/docs/system-prompt-composition.md` — system-prompt composition findings.
