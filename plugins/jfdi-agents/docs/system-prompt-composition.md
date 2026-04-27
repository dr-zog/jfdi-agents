# System-prompt composition for jfdi-agents agents

**Curated reference for prompt authors.** This document is the part you need to read every time you write or audit an agent body.

The three SKILL.md files that point at this doc:

- `skills/write-team-lead/SKILL.md`
- `skills/write-agent/SKILL.md` (used by the Architect to mint Developer agents at architecture time — and by the human if they want to tweak one)

Keep this doc and those skills in sync — when a finding changes, update here and the skill bodies pick it up by reference.

## The short story

The plugin ships a custom output style at `${CLAUDE_PLUGIN_ROOT}/output-styles/jfdi-agent.md` with `keep-coding-instructions: false`. The bootstrap skill writes `"outputStyle": "jfdi-agent"` into `.claude/settings.json` automatically.

This applies to **every agent in the project** — main-session TeamLead and teammates alike — and strips the coding-focused default content (`# Doing tasks`, `# Committing changes with git`, `# Creating pull requests`, the trailing *"do NOT write report/summary/findings/analysis .md files"* Notes block) from every agent's system prompt while retaining Claude Code's agent-harness behaviour.

Approximately 3.5 KB stripped out of a typical 27 KB agent prompt.

## What is retained

The output style strips coding discipline; everything else from the Claude Code base is retained:

1. Security preamble, `# System`, `# Executing actions with care`, `# Using your tools`, `# Tone and style`, `# auto memory`.
2. `# Environment` — cwd, git status, model info.
3. `# Agent Teammate Communication` — *"you are running as an agent in a team; use SendMessage; the user interacts primarily with the team lead"* (teammates only).
4. `# Custom Agent Instructions` boundary heading — harness-injected.
5. The agent body.
6. Session-specific guidance — text-output cadence, session-spawning heuristics.
7. `<system-reminder>` delivery — deferred tools, MCP instructions, skills list, CLAUDE.md, user MEMORY.md.

## What is excluded by `keep-coding-instructions: false`

- `# Doing tasks` — the coding-focused discipline (prefer editing existing files, don't add features beyond the task, default to no comments, no error handling for impossible scenarios, etc.).
- `# Committing changes with git`.
- `# Creating pull requests`.
- `# Other common operations`.
- The trailing Notes block, including *"Do NOT Write report/summary/findings/analysis .md files"*.

The output style also adds a short framing line at the top, introducing the team context without prescribing the workflow details (those come from the agent body and the shared docs).

## Consequences for prompt authoring

### (a) Markdown-authoring agents need NO counter-prime

The coding-focused ambient that used to misalign with their role — *"prefer editing existing files"*, *"don't add features beyond the task"*, *"default to writing no comments"*, and critically *"do NOT write report/summary/findings/analysis .md files"* — is **not present**. Bodies can focus on identity / competence / deliverables / hard-boundaries without having to countermand base-prompt directives.

Markdown-authoring roles in this plugin: `TeamLead` (status blocks only — no disk writes), `ProductOwner`, `Architect` (for architecture.md, decisions.md, and the `.claude/agents/*-dev.md` files it mints), `Verifier` (demo reports). These bodies are short and principled — no counter-prime section needed.

### (b) Code-writing agents add coding discipline explicitly

The Architect-minted `<layer>-dev` agents genuinely write production code, and `Verifier` occasionally writes throw-away diagnostic code (never committed). With the output style stripping the ambient default, these bodies include the discipline as positive prose along the lines of:

> *"You write code. Stay within your folder: `<folder>/`. Follow the project's `docs/architecture.md` for language/framework conventions and the Technology stack section for pinned dependencies. Beyond that, apply these general heuristics: prefer editing existing files over creating new ones; match existing style; don't add features beyond the acceptance item; default to no comments unless the WHY is non-obvious; no error handling for scenarios that can't happen; validate only at system boundaries; three similar lines is better than a premature abstraction. Commit incrementally."*

Opt-in rather than ambient. That is a feature: the role owns its own discipline and doesn't silently inherit it. The `write-agent` skill bakes this into every minted developer body.

### (c) CLAUDE.md still applies regardless

CLAUDE.md is delivered via `<system-reminder>` to every agent regardless of output-style setting. Project-wide invariants — branching discipline, commit authorship, etc. — go there.

## What the agent body must still carry

The output style handles generic prime; it does not handle role or plugin specifics. Every agent body still needs:

- **Identity and mission** — who the agent is and what it does.
- **Read-these-first pointers** — the shared docs the agent needs (`docs/terminology.md`, its roster entry, etc.). Use `${CLAUDE_PLUGIN_ROOT}/docs/...` form so paths resolve in both dev-mode and installed mode.
- **Competence / priors** — the expertise the agent brings, what it optimises for, internal standards.
- **Deliverables** — file paths, formats, quality bar.
- **Hard boundaries** — what the agent refuses to do, and why. Prefer mechanical enforcement (`disallowed-tools` frontmatter) where possible.
- **The `team-lead` addressing rule** wherever `SendMessage` to the lead is referenced. The base does not distinguish role name from addressable name.
- **Commit authorship pointer** — one line naming the `--author=` flag and the `.invalid` email slug. Not in the base; role-specific.
- **Folder ownership (developers only)** — the single folder the agent may edit. Hard boundary.

For the TeamLead body, additionally:

- The state machine — stages the lead sequences, prerequisites, completion criteria.
- Spawn briefs — runtime templates per teammate.
- Wait semantics, human-escalation fallback, and teardown mechanics.

## What does NOT reach teammates

- The `/agents` list, the plugin list, and other discovery surfaces specific to the main session.
- The ability to set a *different* output style per-teammate — the output style is project-level, not per-agent. If a specific role needs a different composition, that goes in the body.

## The special case: Architect-minted developer agents

Developer agents live in `.claude/agents/` in the **downstream project**, not in the plugin. This has two consequences:

1. **They are project-scoped subagents.** Claude Code resolves them from the project's `.claude/agents/` directory on each spawn. Edits take effect immediately — no plugin cache to bust, no version bump needed.
2. **They still pick up the output style.** `.claude/settings.json`'s `outputStyle: "jfdi-agent"` applies project-wide; the minted agents get the same stripped base prompt as the plugin-shipped agents.

The `write-agent` skill (`skills/write-agent/SKILL.md`) bakes in the frontmatter fields these agents need — `name`, `description`, `tools`, `disallowed-tools: AskUserQuestion` — plus the body template described in the roster. When the Architect invokes the skill, it produces a well-formed agent file that the TeamLead can spawn immediately.

## Diagnostic — confirming the output style is active

If you suspect an agent is running without the output style (misconfigured project, stale settings), the diagnostic is to check the agent's context for `# Doing tasks` or the *"do NOT Write report/summary/findings/analysis .md files"* Notes directive. If either is present, the output style is NOT active and the agent is running with ambient coding defaults — surface the issue to the human rather than silently proceeding.

## Why this matters

Before the output-style resolution, every markdown-authoring agent body carried an explicit counter-prime block: *"You write reports/briefs/reviews. The base prompt's directive against writing .md files does not apply to you."* Six bodies, six near-identical counter-primes, one source of drift.

After: zero counter-primes. The output style strips the directive at source. Bodies focus on identity and competence; the system-prompt composition is uniform and predictable.

This is exactly the kind of finding that drifts silently if duplicated. Single source of truth here; single pointer from each skill body.
