# Terminology

> **This document is canonical.** Every agent in this plugin speaks this language. If an agent's system prompt or a user's natural-language request contradicts the definitions here, the definitions here win. If a word is missing from this document, it is not yet part of the team's vocabulary — propose it here before using it.

## Why a shared vocabulary?

A six-role team that can't agree on what a "layer" is, or whether "developer" means a code-writing agent or a human collaborator, will pass work between its members with subtle meaning drift. This document pins the words so that when the Architect says *"the backend-dev owns the `backend/` folder"*, every other agent knows exactly which agent, which folder, and what that ownership implies.

## The JFDI posture

The plugin's name is deliberate. **Just F***ing Do It** is a reaction to prior experiments — spec-first and ATDD-style agent teams that bogged down in up-front ceremony (roadmaps, cycles, feature briefs, scenario Gherkin, three-amigos meetings) before producing a single line of working code. This plugin's core bet is that a small, architect-led team, building a walking skeleton one layer at a time before ever parallelising, will reach *"we have a running system"* faster and with less drift.

Documentation is kept deliberately light:

- A **Vision** the Product Owner authors.
- An **acceptance list** — numbered one-line requirements, not Gherkin.
- A short **architecture** doc naming the layers, languages, and folder map.
- A short **decisions log** for calls that would otherwise get re-litigated.
- **Demo reports** — one per walking-skeleton layer, one per refinement pass.

That is all. No roadmap, no cycles, no feature files, no step definitions, no session notes (TeamLead may leave continuity notes if it helps; nobody else).

## The team's shape

```
TeamLead              (conductor, reads everything, writes nothing)
   ├── ProductOwner   (owns Vision and acceptance list)
   ├── Architect      (owns architecture, folder map, mints developer agents, resolves technical disputes)
   ├── Developers     (minted at architecture time — one per layer — each owns one folder)
   ├── Verifier       (runs tests, checks acceptance list, writes demos)
   └── RepoSteward    (branch lifecycle — no content writes)
```

Six standing roles. The **Developers** row is unusual: it is not a single agent but a *family of agents minted at architecture time* by the Architect using the `write-agent` skill. Each developer is tailored to one layer's language stack and owns exactly one folder in the downstream repo's tree. The number of developers a project has (typically 1–4) is a decision, not a fixed number.

## Core concepts

### Vision

**Definition.** The product's reason for existing — what problem it solves, who it is for, what is explicitly in and out of scope, what success looks like. Also declares the product's **architectural components** (persistence, backend service, user frontend, admin frontend, CLI, etc.) and the **code review platform** (`github | gitlab | bitbucket | none`).

**Owner.** `ProductOwner`.

**Artefact.** `vision/` directory in the downstream user's repo:
- `vision/overview.md` — elevator pitch, problem, users
- `vision/goals.md` — success criteria and non-goals
- `vision/constraints.md` — architectural components list, code review platform, regulatory/commercial/technical constraints
- `vision/personas.md` — end-user and operator personas
- `vision/glossary.md` — user-facing vocabulary (distinct from this document)

**Cadence.** Created during the intake interview. Revised only when the product's reason for existing genuinely changes.

### Architecture

**Definition.** The Architect's statement of how the Vision's components are realised — languages chosen, layers named, folder map committed, non-obvious decisions recorded.

**Owner.** `Architect`.

**Artefact.** `docs/architecture.md` — short, declarative, skimmable. Must contain:
- A **Layers** section naming each layer (e.g. `data`, `backend`, `frontend`, `shared`), the language chosen for each, and a one-paragraph justification.
- A **Folder map** section binding each layer to a top-level folder in the repo (e.g. `backend/` → backend-dev).
- A **Developer roster** section listing the minted developer agents, naming each one's layer, language stack, and owned folder.
- A **Technology stack** section listing pinned dependencies the team commits to.

Pairs with `docs/decisions.md` — an append-only one-line log for non-obvious architectural calls.

### Acceptance list

**Definition.** The numbered list of end-user-observable requirements the project commits to deliver. Prose, one line each, written from an end-user's perspective. No Gherkin, no Given/When/Then, no implementation detail. The Verifier tests against this list.

**Owner.** `ProductOwner` + `Architect` jointly — PO drives wording, Architect checks technical realism.

**Artefact.** `vision/acceptance.md`.

**Examples.**
- *A user can add a task by typing it and pressing Enter; the task appears in their list.*
- *An admin can reset a user's password; the user receives an email with a reset link.*
- *Tasks persist across app restarts.*

**Cadence.** Drafted in the architecture stage. Extended (not rewritten) as the product grows — each new acceptance line gets the next number, with a date in parentheses.

### Layer

**Definition.** A named logical slice of the system that one developer agent owns. Layers are the unit the Architect uses to decompose the system into folder-per-layer ownership. Typical layers:

| Layer | Folder (default) | Typical language choices |
|---|---|---|
| data | `data/` or `db/` | SQL migrations, ORM schemas, seed scripts |
| backend | `backend/` or `api/` | TypeScript/Node, Python/FastAPI, Go, Ruby on Rails |
| middleware | `middleware/` or `bff/` | Same as backend, usually thinner |
| frontend | `frontend/` or `web/` | React, Svelte, Vue, HTMX |
| shared | `shared/` or `packages/shared/` | Common types, utilities, constants — used by multiple layers |
| mobile | `mobile/` | React Native, Swift, Kotlin |
| cli | `cli/` | Go, Rust, Node |

The Architect decides which layers a given project has. Not every layer is present; a library has one, a full-stack web app typically has three or four.

### Folder map

**Definition.** The committed binding of layers to top-level repo folders. Each developer agent owns exactly one folder; developers do not cross folder boundaries. The Architect's `docs/architecture.md` records the map; the developer agent bodies include their owned folder as a hard constraint.

**Why this matters.** Folder ownership replaces the much heavier coordination machinery of feature-branching-per-story. Two developers can work in parallel without coordination *because their folders don't overlap*. When they do need to coordinate (e.g. the backend changes an API shape the frontend consumes), the Architect brokers.

**Shared code.** If two layers would need the same type or utility, the Architect adds a `shared/` layer with its own developer, and those types/utilities live there.

### Developer role

**Definition.** An Architect-minted agent that owns one layer's folder, writes code in that layer's language, and does not touch other folders. The Architect writes the developer's agent definition into `.claude/agents/<role-name>.md` during the architecture stage.

**Owner.** `Architect` authors; `TeamLead` spawns instances at build/refine time.

**Naming convention.** `<layer>-dev` (e.g. `backend-dev`, `frontend-dev`, `data-dev`, `shared-dev`). When spawned as a teammate, the instance gets a suffix: `backend-dev-skeleton`, `backend-dev-refine-3`. Lowercase kebab-case throughout.

**Artefact.** `.claude/agents/<role-name>.md` in the downstream project, authored by the Architect via the `write-agent` skill. See `${CLAUDE_PLUGIN_ROOT}/skills/write-agent/SKILL.md`.

**Key constraints baked into every developer body.**
- *You own one folder. Do not edit files outside it.* (The folder is named in the body.)
- *You write in one language stack.* (The stack is named in the body.)
- *If you need something from another layer that does not yet exist, ask the Architect via the TeamLead relay. Do not reach across.*

### Walking skeleton

**Definition.** The first end-to-end build of the system — the thinnest possible path through every declared layer that a user can exercise. Alistair Cockburn's term, also known as a *tracer bullet* (Hunt & Thomas). The plugin's first phase of real work produces a walking skeleton.

**Discipline.** Built **sequentially**, not in parallel. The Architect walks one developer at a time down the stack: data first, then the layer above, then the layer above that. Each developer's contribution assumes the layers below it and leaves stubs for the layers above. The skeleton is **running** before the second phase begins.

### Build phase

**Definition.** The sequential walking-skeleton construction that follows architecture. The Architect is an active participant — they shepherd one developer at a time, verify the layer works before advancing, and commit each layer's work as its own milestone.

**Stops when.** The walking skeleton exercises every acceptance-list item at least trivially (even if shallowly), the Verifier runs the full acceptance list and produces a green-ish demo, and the Architect signs off.

### Refine phase

**Definition.** The parallel-work phase that follows build. The TeamLead can spawn multiple developers simultaneously because each owns a different folder. Typical refine work: acceptance items that were stubbed, bug fixes, polish, additional acceptance items discovered during use.

**Discipline.** Each developer stays in its own folder. Cross-folder coordination routes through the Architect. The Verifier runs after each batch of parallel work.

### Demo

**Definition.** The Verifier's record of what was built, what runs, what each acceptance item's status is, and any findings. Written at the end of each Build milestone and after each Refine pass.

**Owner.** `Verifier`.

**Artefact.** `docs/demos/<YYYY-MM-DD>-<slug>.md` in the downstream project. Slug examples: `skeleton-data-layer`, `skeleton-complete`, `refine-pass-1`, `bugfix-auth-reset`.

**Contents.**
- The command(s) run to start the system (verbatim)
- Acceptance-list status table (green / red / not-yet-exercised, per item)
- Three-axis audit: completeness, correctness, coherence
- Findings classified CRITICAL / WARNING / SUGGESTION
- Ready-to-advance: Yes | Not yet — drives the TeamLead's next stage decision

### Decisions log

**Definition.** Append-only one-line record of non-obvious technical calls the Architect makes. Replaces the heavier ADR tradition — sized for a small team moving fast.

**Owner.** `Architect`.

**Artefact.** `docs/decisions.md`.

**Format.** One Markdown line per decision:

```
- **2026-04-22** — Pinned Postgres 16. SQLite considered; rejected because acceptance #7 requires concurrent writes.
- **2026-04-23** — Frontend bundler: Vite. No strong reason; default for the stack.
```

A decision gets logged when a future reader (human or agent) might otherwise ask *"why did we pick this?"* and have no answer. Obvious defaults (use npm for JavaScript, use pytest for Python) do not need entries.

## Stages

The team moves through four stages. Stages are scope-boxed, not time-boxed — a stage completes when its artefacts exist and are signed off.

### Stage 1 — Intake

**Who.** TeamLead + ProductOwner.

**Output.** `vision/` populated.

**Discipline.** Narrow questions, one at a time, relayed via TeamLead's `AskUserQuestion`. ProductOwner does not speculate about architecture.

### Stage 2 — Architecture & team design

**Who.** TeamLead + ProductOwner + Architect.

**Outputs.**
- `docs/architecture.md` with Layers, Folder map, Developer roster, Technology stack sections
- `docs/decisions.md` seeded with the initial pinning calls
- `vision/acceptance.md` — the numbered acceptance list, jointly authored
- `.claude/agents/<layer>-dev.md` — one file per developer the Architect minted, authored via the `write-agent` skill

**Discipline.** The Architect's first job is to decide the layer list. The second is to decide which layers need their own developer (a layer can live in another dev's folder if both are the same language and the Architect judges it simpler). The third is to mint the developer agents. The fourth is to co-author the acceptance list.

**Before leaving this stage.** The TeamLead verifies all developer agents are on disk and parseable. If something's missing, the Architect finishes it before Build starts.

### Stage 3 — Build (walking skeleton)

**Who.** TeamLead + Architect + one Developer at a time + Verifier at layer boundaries.

**Flow.** Architect picks the lowest layer (typically data or a stub-heavy shared layer). TeamLead spawns the corresponding developer as a teammate. Architect guides, developer commits. When the layer is complete, TeamLead spawns Verifier to run whatever acceptance items are meaningful at that slice. On green, advance to the next layer.

**Stops when.** The top layer is in place, the system starts end-to-end, and Verifier runs a full acceptance-list demo with a majority of items green or documented-as-deferred.

### Stage 4 — Refine (parallel work)

**Who.** TeamLead + any number of Developers in parallel + Verifier between passes + Architect on call.

**Flow.** TeamLead identifies work that can be parallelised (each acceptance item touches one or two folders). Spawns the relevant developers simultaneously. Verifier runs after each pass. Architect is summoned for cross-folder contract decisions.

**Stops when.** The acceptance list is fully green, or the human says stop.

## Personas

The plugin uses two personas in acceptance-list wording:
- **end-user** — the human (or caller) the product exists to serve.
- **operator** — the human who installs, runs, observes, upgrades, or debugs the product.

Both personas are seeded in `vision/personas.md`. When a Developer or Verifier needs to judge a wording question *"would an operator understand this error message?"*, they `SendMessage` the ProductOwner with the question and the persona tag.

If a project has no distinct operator (pure library, static site), ProductOwner records that explicitly.

## Words we deliberately avoid

These words are banned from team communication because they are ambiguous, overloaded, or borrowed from methodologies this plugin deliberately departs from:

- **sprint** — time-boxed; we are scope-boxed. Use *stage* or *pass*.
- **epic** / **story** / **ticket** — story-point baggage; use *acceptance item* or name the work directly.
- **iteration** — ambiguous; use *refine pass* when that's what you mean.
- **cycle** / **feature** (as a planning unit) / **scenario** / **step** — vocabulary of the retired ATDD variant; do not use.
- **roadmap** — the plugin has no Planner. If the user asks for a roadmap, relay it to the ProductOwner who captures it as extensions to the acceptance list.
- **MVP** — allowed only in Vision prose. Do not name an artefact "MVP".
- **test** — allowed in casual prose but be specific: *acceptance item*, *unit test*, *integration test*, *the suite*.
- **backend-first / frontend-last / layer-by-layer** — violates the walking-skeleton discipline. Data first is still thin-slice-first; every layer exists in Stage 3 before Stage 4 begins.
- **three-amigos** — we do not hold three-amigos meetings. The Architect talks to one developer at a time during Build and brokers between developers during Refine.
- **gherkin** / **cucumber** / **feature file** — the plugin does not use Gherkin. Acceptance items are prose.

If a human uses one of these words, agents should translate in the response but not correct the human unless the ambiguity is blocking.

## Words with deliberate overlap

These words appear in both the Claude Code surface and our vocabulary. The overlap is intentional; context disambiguates:

- **Task** — Claude Code's TaskCreate/TaskList tool identifier. Do not overload with a planning meaning.
- **Skill** — a Claude Code [Skill](https://code.claude.com/docs/en/skills). We never use "skill" to mean "capability" in a specification sense.
- **Agent** — a Claude Code subagent or team member. We never use "agent" to mean "autonomous process" in the generic sense.
- **Layer** — *our* layer is the folder-ownership unit. "Architectural layer" (DDD, OSI) is a related but broader concept; qualify when it matters.
- **Component** — the Vision's *architectural components* list is one use; a React *component* is another. Qualify if ambiguous.
- **Developer** — *our* Developer is a minted agent role. The *human developer* (i.e. the user of this plugin) is the TeamLead's counterpart in all interactive conversations.

## Glossary quick-reference

| Term | Owner | Where it lives |
|---|---|---|
| Vision | ProductOwner | `vision/` |
| Architectural components | ProductOwner (in `constraints.md`) | `vision/constraints.md` |
| Code review platform | ProductOwner (in `constraints.md`) | `vision/constraints.md` |
| Acceptance list | ProductOwner + Architect | `vision/acceptance.md` |
| Architecture | Architect | `docs/architecture.md` |
| Layers | Architect (within architecture.md) | `docs/architecture.md` |
| Folder map | Architect (within architecture.md) | `docs/architecture.md` |
| Decisions log | Architect | `docs/decisions.md` |
| Developer agents | Architect (authors); TeamLead (spawns) | `.claude/agents/<layer>-dev.md` |
| Demo report | Verifier | `docs/demos/<date>-<slug>.md` |
| Merge commits | RepoSteward | `git log --merges` |
| Feature branches | RepoSteward | `git branch --list` |
