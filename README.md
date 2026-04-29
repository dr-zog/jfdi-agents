# jfdi-agents

> A Claude Code plugin containing a six-role agent team that autonomously designs and builds software using the **JFDI pattern** — an architect-led, sequential-then-parallel approach where the Architect mints per-layer developer agents tailored to the project's chosen stack.

**Status:** pre-release. Interfaces, names, and behaviour may change.

## What is this?

`jfdi-agents` is a Claude Code plugin that turns an empty repo into a fully-staffed virtual product team. You install the plugin, sit down for a scoping interview with the Product Owner, and from there the team can carry a product from vision to running code with as little human involvement as you want.

It is a deliberate reaction to its predecessors (spec-first and ATDD-driven agent teams) which bogged down in up-front ceremony — roadmaps, cycles, feature briefs, Gherkin scenarios, three-amigos meetings — before producing a single line of working code. JFDI stands for **Just F\*\*\*ing Do It**: ship a running walking-skeleton as quickly as coherent process allows, then parallelise refinement.

### The pattern, in one paragraph

ProductOwner runs a Vision intake. Architect reads the Vision, decides the layers (data, backend, middleware, frontend, shared — whatever the system actually needs), picks one language stack per layer, commits to a folder map, and **mints one developer agent per layer into `.claude/agents/`** using the plugin's `write-agent` skill. PO + Architect jointly author a numbered acceptance list. The team then walks the stack **sequentially**, one developer at a time, building the thinnest possible end-to-end walking skeleton. Only after Verifier signs off on the skeleton does the team parallelise — multiple developers working simultaneously in their own folders, refining the product against the acceptance list.

### Why this shape?

- **Folder-per-developer ownership.** Each minted developer owns exactly one folder. Parallelism is safe because folders don't overlap. Coordination routes through the Architect.
- **Sequential skeleton, parallel refinement.** Parallelism before the skeleton exists is the single most common failure mode of autonomous agent teams — three "done" layers that don't compose. JFDI forbids it.
- **Minted per-project developers.** The developer agents aren't generic — they're tailored to the project's chosen languages, pinned dependencies, and folder map. The Architect writes them during architecture. The human can tweak them.
- **Light documentation.** Vision, short architecture doc, numbered acceptance list, append-only decisions log, demo reports. No roadmap, no cycles, no features, no Gherkin, no session notes.

## Quick start

**Step 1 — Install the plugin.** From your terminal, `cd` into the project directory where you want the workflow to run (an empty directory is fine), start a Claude Code session, and run the two install slash-commands:

```bash
cd /path/to/your/project
claude
```

Then inside the session, add the marketplace (substitute your own remote URL for a fork):

```
/plugin marketplace add https://github.com/dr-zog/jfdi-agents.git
```

Then install the plugin:

```
/plugin install jfdi-agents@jfdi-agents
```

When the install runs, Claude Code prompts you to pick a scope: **User**, **Project**, or **Local**. **Recommended: Project** — it writes the plugin entry to `./.claude/settings.json` alongside the project itself, so anyone who clones the repo and trusts the folder gets the same plugin set-up.

After the install completes, reload plugins so the new skills are visible:

```
/reload-plugins
```

**Step 2 — Run the bootstrap pre-flight.** Still in the same install session, run:

```
/jfdi-agents:bootstrap
```

This is the plugin's pre-flight skill. It checks git, initialises a repo if needed, writes the required env vars to `.claude/settings.json` at project scope (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` and `CLAUDE_CODE_DISABLE_1M_CONTEXT=1`), sets the `jfdi-agent` output style, and lays down a deliberately-small directory tree (`vision/`, `docs/`, `docs/demos/`, `.claude/agents/`). Idempotent; safe to re-run. No frameworks or test runners are chosen here — the Architect makes those decisions during Stage 2, once the Vision is known.

**Step 3 — Exit this session and launch the TeamLead in a fresh one.** Claude Code picks its main-thread agent at session launch, so you need a new session with TeamLead as its main thread:

```
/exit
```

Then from your terminal, still in the same project directory:

```bash
claude --agent jfdi-agents:team-lead
```

**Step 4 — Tell the TeamLead to get started.** Claude Code sessions start idle — the TeamLead's system prompt is loaded but it won't do anything until you prompt it. Paste this to kick off the default Checkpointed-mode workflow:

```
Proceed in Checkpointed mode. Run the Vision intake interview with me, then drive through architecture & team design, the walking-skeleton build, and refinement. Pause for my approval after each load-bearing stage.
```

The TeamLead then creates the agent team (after asking your confirmation), runs its state survey, and — because bootstrap has already run — spawns ProductOwner as the first interactive specialist for the Vision intake. From there you answer the interview questions and can walk away between checkpoints.

For an unattended run — team drives all the way to a green acceptance list without pausing — swap in this kickoff prompt instead:

```
Run in Autonomous JFDI mode. Drive the Vision intake with me (autonomy begins once Vision is captured), then continue through architecture, the walking-skeleton build, and refinement until the acceptance list is fully green. Stop only on a specialist blocker. Remember: Autonomous JFDI means every rule still runs — sequential-skeleton rule, folder-ownership rule, Verifier sign-off between stages. No shortcuts.
```

**Prerequisites:**

- Claude Code v2.1.32 or later — [agent teams](https://code.claude.com/docs/en/agent-teams) are a hard requirement.
- `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` and `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` in Claude Code settings — the Step 2 `/jfdi-agents:bootstrap` skill auto-writes both to `.claude/settings.json` (project scope) if missing, so you don't need to set them manually.

Full install details (updating, uninstalling, dev mode for contributors, private-repo auth) in [`plugins/jfdi-agents/docs/local-install.md`](plugins/jfdi-agents/docs/local-install.md).

## Per-project isolation (strongly recommended)

Claude Code stores **all** state — settings, credentials, plugins, session history, and the agent-teams directory — under `CLAUDE_CONFIG_DIR` (default `~/.claude`). Every project on your machine shares that directory by default.

This is a problem for this plugin specifically: the TeamLead's zombie-recovery routine is destructive (`rm -rf` of a team directory) and operates under whatever `CLAUDE_CONFIG_DIR` points at. A recovery against a mistakenly-identified team name in the shared `~/.claude/teams/` tree can wipe a team belonging to a different project.

The fix is one environment variable. Scope a session's entire state tree to the project by setting `CLAUDE_CONFIG_DIR` before launching:

```bash
cd /path/to/your/project
export CLAUDE_CONFIG_DIR="$PWD/.claude-state"
mkdir -p "$CLAUDE_CONFIG_DIR"
claude --agent jfdi-agents:team-lead
```

Or as a shell alias:

```bash
alias claude-jfdi='CLAUDE_CONFIG_DIR="$PWD/.claude-state" claude'
```

With `CLAUDE_CONFIG_DIR` scoped, cross-project collisions are structurally impossible. The TeamLead's zombie recovery can only ever touch this project's state tree.

**Caveats:**

- First launch in a fresh `CLAUDE_CONFIG_DIR` wants its own `claude auth login` and `/plugin install jfdi-agents@jfdi-agents`. The isolation is stronger at the cost of a per-project setup.
- `.claude-state/` contains credentials — the bootstrap skill adds it to your `.gitignore` automatically. If you rename the directory, make sure your own `.gitignore` still catches it.

**If you don't set `CLAUDE_CONFIG_DIR`**, the bootstrap skill warns you at setup, the TeamLead warns you at session start, and any destructive zombie recovery requires explicit human confirmation first. But the safe-by-default posture is to scope every session.

## The roster

| # | Agent | One-line role |
|---|-------|---------------|
| 1 | **TeamLead** | Conducts the workflow autonomously — never builds, only sequences |
| 2 | **ProductOwner** | Owns the Vision and the numbered acceptance list |
| 3 | **Architect** | Architecture, folder map, decisions log; mints per-layer developer agents; resolves technical disputes |
| 4 | **Developer** (minted) | A *family* of Architect-minted agents, one per layer, each owning one folder |
| 5 | **Verifier** | Runs the system, checks the acceptance list, writes demo reports |
| 6 | **RepoSteward** | Branch lifecycle — create, checkout, merge, delete |

Full descriptions, owned artefacts, and handoff rules live in [`plugins/jfdi-agents/docs/roster.md`](plugins/jfdi-agents/docs/roster.md).

## The shared vocabulary

All agents speak the same language about planning, which avoids the ambient confusion of "is this a stage, a milestone, a sprint, or a story?". The canonical stages and artefacts are:

```
Intake → Architecture & team design → Build (walking skeleton) → Refine (parallel)
```

And the nouns:

- **Vision** — the product's north star, owned by ProductOwner
- **Layers** — the logical slices the Architect decides on (data, backend, frontend, shared, …)
- **Folder map** — the binding of layers to top-level repo folders
- **Acceptance list** — numbered, prose, end-user-observable
- **Walking skeleton** — the first end-to-end build through every layer, built sequentially
- **Refine pass** — a parallel-work batch after the skeleton exists

Full definitions in [`plugins/jfdi-agents/docs/terminology.md`](plugins/jfdi-agents/docs/terminology.md).

## How it runs

```
1. /plugin marketplace add <this-repo-url>
2. /plugin install jfdi-agents@jfdi-agents
3. /jfdi-agents:bootstrap
4. /exit
5. claude --agent jfdi-agents:team-lead
6. "Proceed in Checkpointed mode..."
7. Walk through the Vision intake. Answer questions that appear in the TeamLead session.
8. After architecture: review the layer list and the acceptance list; approve or redirect.
9. Walk away during Build. Come back to each layer's demo.
10. After skeleton-complete: acceptance list runs end-to-end.
11. Refine passes close the remaining gaps.
```

The **TeamLead** is the single main-session agent that drives the whole workflow. At session start, it asks you to confirm creation of a Claude Code agent team (per Anthropic's [agent teams docs](https://code.claude.com/docs/en/agent-teams), only you can authorise team creation). Confirm and the TeamLead becomes the team lead for the rest of the session. It then spawns specialist teammates as the workflow requires — ProductOwner first (for the Vision intake, via the relay pattern), then Architect alongside ProductOwner for Stage 2, then per-layer developers during Build, then multiple developers in parallel during Refine.

**The relay pattern.** The TeamLead is the only agent that talks to you directly. Specialists (ProductOwner, Architect, the minted Developers, Verifier, RepoSteward) have `AskUserQuestion` disallowed at the frontmatter level — they cannot ask you anything on their own. When a specialist needs your input, it `SendMessage`s the TeamLead with a question brief; the TeamLead presents the question to you via `AskUserQuestion` and relays your answer back.

**Checkpointed mode** (default) pauses for your approval after every load-bearing transition — post-intake, post-architecture, per Build-layer demo, post-skeleton-complete, per Refine-pass demo. **Autonomous JFDI mode** runs straight through without pausing; opt in explicitly when you want it. The mode names are deliberate: Checkpointed means "I'll catch shortcuts"; Autonomous JFDI means "you can't catch them, so don't take any". Every rule (sequential-skeleton, folder-ownership, Verifier sign-off) remains non-negotiable in Autonomous JFDI — the lack of a human checkpoint makes the process rigour *more* important, not less.

**There is only one supported invocation path.** `claude --agent jfdi-agents:team-lead` creates the team and drives everything inside it. Per-role main-session launches are **not supported** — the specialists cannot function outside the TeamLead-led team because `AskUserQuestion` is disallowed and their prompts assume the relay.

## The minted developers

This is the plugin's distinguishing feature and worth calling out on its own. During Stage 2, the Architect picks languages + a folder map and then authors one agent definition per layer, written to the downstream project's `.claude/agents/` directory. After Stage 2, a typical repo contains:

```
.claude/agents/
├── data-dev.md
├── backend-dev.md
├── frontend-dev.md
└── shared-dev.md     # if the Architect declared a shared layer
```

Each file is a well-formed Claude Code subagent with:

- The layer's chosen language stack baked into the body.
- A **folder ownership** hard boundary (`"You own backend/. Do not edit files outside it."`).
- The coding-discipline clause (prefer editing, match existing style, don't add features beyond the acceptance item, no error handling for impossible cases, etc.) — since the plugin's output style strips Claude Code's default coding prompt, it has to be positive prose inside the body.
- A cross-folder-coordination protocol routing to the Architect.

You can tweak these files by hand if the Architect's defaults don't suit you — they're just Markdown. The `/jfdi-agents:write-agent` skill is also available to regenerate one with different inputs.

## Repository layout

This repo is a **one-plugin Claude Code marketplace**. The marketplace catalog lives at the repo root (`.claude-plugin/marketplace.json`), and the plugin itself lives one level down at `plugins/jfdi-agents/`.

```
jfdi-agents/                             # repo root = marketplace root
├── .claude-plugin/
│   └── marketplace.json                 # marketplace catalog (one entry)
├── plugins/
│   └── jfdi-agents/                     # plugin root
│       ├── .claude-plugin/
│       │   └── plugin.json              # plugin manifest
│       ├── agents/                      # five specialist definitions
│       │   ├── team-lead.md
│       │   ├── product-owner.md
│       │   ├── architect.md
│       │   ├── verifier.md
│       │   └── repo-steward.md
│       ├── skills/
│       │   ├── bootstrap/SKILL.md          # /jfdi-agents:bootstrap
│       │   ├── start/SKILL.md              # /jfdi-agents:start
│       │   ├── commit-as-agent/SKILL.md    # referenced by every file-writing agent
│       │   ├── write-team-lead/SKILL.md    # meta-skill for authoring lead prompts
│       │   └── write-agent/SKILL.md        # meta-skill for authoring specialist and dev prompts
│       ├── output-styles/
│       │   └── jfdi-agent.md               # strips Claude Code's coding default
│       ├── scripts/
│       │   └── stall-detector.sh           # background process the TeamLead monitors
│       └── docs/
│           ├── terminology.md              # shared vocabulary (all agents cite this)
│           ├── roster.md                   # agent scopes and handoffs
│           ├── process.md                  # collaboration, conflict, commit rules + artefact templates
│           ├── team-lead-playbook.md       # team-management mechanics (lead satellite of process.md)
│           ├── solo-agents.md              # solo-mode reference (solo satellite of process.md)
│           ├── system-prompt-composition.md # output-style rationale
│           └── local-install.md            # install via git URL, dev mode, etc.
├── evals/                                  # end-to-end evals against the plugin
├── tools/                                  # diagnostic tools (team-inspect Go binary)
├── CLAUDE.md                               # repo-level instructions for Claude Code
├── LICENSE                                 # MIT
├── NOTICE.md                               # third-party attributions
└── README.md                               # this file
```

**The `${CLAUDE_PLUGIN_ROOT}` substitution in agent bodies** always resolves to the plugin's *installed* root, which is either `<cache-dir>/jfdi-agents-jfdi-agents/<version>/` (installed mode) or `<your-checkout>/plugins/jfdi-agents/` (dev mode). Either way, `${CLAUDE_PLUGIN_ROOT}/docs/terminology.md` lands at the correct file inside the plugin.

## Contributing

Architectural changes to the plugin itself should go through the plugin's own workflow once the eval suite stabilises — dog-fooding the team on its own source of truth.

## Licence

MIT. See [`LICENSE`](LICENSE) for the full text and [`NOTICE.md`](NOTICE.md) for third-party attributions.
