---
name: bootstrap
description: One-stop pre-flight for starting a new project with the jfdi-agents plugin. Run this as the first thing after installing the plugin. Checks prerequisites (git), initialises a git repo if needed, writes the required Claude Code project settings to `.claude/settings.json` (the agent-teams env var, the 1M-context disable, and `outputStyle: "jfdi-agent"` — the load-bearing piece for system-prompt composition across every agent), creates the shared directory structure (`vision/`, `docs/`, `docs/demos/`, `.claude/agents/`), seeds placeholder READMEs. Idempotent — safe to re-run; existing files are left alone. Use this skill whenever a user installs the plugin for the first time, asks "how do I set up jfdi-agents in this repo?", or needs to repair a partially-bootstrapped project. After this completes, the project is ready for `claude --agent jfdi-agents:team-lead`.
---

You are bootstrapping a new project for the `jfdi-agents` team. This skill is the human's one-stop prep command — it gets everything in order so the TeamLead can launch and run without bouncing the human back out for environment fixes. Idempotent — if something is already in place, leave it alone. If something is missing, create it. Never overwrite existing files unless you are explicitly merging a known JSON structure (like `.claude/settings.json`).

## Context about the plugin

This plugin ships a six-role agent team (see `agents/` inside the plugin install directory) that drives software development using the JFDI pattern: Product Owner + Architect design a layered system, Architect mints per-layer developer agents, the team walks the stack sequentially for a walking skeleton, then parallelises refinement. The team operates on a specific directory structure in the *user's* project, which this skill creates.

The specific languages, frameworks, and the per-layer developer agents are **not** chosen at bootstrap. The Architect makes those decisions during Stage 2 (Architecture & team design), once the Vision is known. Bootstrap only lays down empty structure.

## Step 1: Check prerequisites

### 1a. Warn if `CLAUDE_CONFIG_DIR` is not scoped to the project

Claude Code stores all state — settings, credentials, plugins, and crucially the agent-teams directory — under `CLAUDE_CONFIG_DIR` (defaults to `~/.claude`). If the human launches `claude` without overriding this, every project on the machine shares the same teams directory. A zombie-team recovery in this project's TeamLead session can then `rm -rf` another project's team state.

Check whether `CLAUDE_CONFIG_DIR` is set **and** points inside the current project:

```bash
if [ -z "$CLAUDE_CONFIG_DIR" ]; then
  echo "WARN: CLAUDE_CONFIG_DIR is unset — Claude Code will use ~/.claude (shared across every project on this machine)."
elif [[ "$(cd "$CLAUDE_CONFIG_DIR" 2>/dev/null && pwd)" != "$PWD"* ]]; then
  echo "WARN: CLAUDE_CONFIG_DIR ($CLAUDE_CONFIG_DIR) is not inside the project ($PWD)."
else
  echo "OK: CLAUDE_CONFIG_DIR is scoped to this project."
fi
```

Print the result as-is. Do not refuse to continue on a warning — the human may have deliberately scoped it differently, or may be running a one-off eval. Record whichever state you observed and include it in the final summary so they can act if they want to.

### 1b. Git repository

The plugin's whole workflow depends on git — feature branches per stage, Verifier sign-off via demo reports, incremental commits. If the current directory is not already a git repo, initialise one:

```bash
git rev-parse --is-inside-work-tree 2>/dev/null && echo "git-ready" || git init
```

If `git init` fails (e.g. no `git` on `PATH`), stop and tell the user:

> `git init` failed — git may not be installed or available in this shell. Please install git and re-run bootstrap.

### 1c. Claude Code project settings — env vars + output style

The `jfdi-agents` plugin needs three entries in the project's `.claude/settings.json`. None of them are needed by bootstrap itself; they take effect when the user launches the TeamLead later. Bootstrap's job is simply to ensure they are present.

**Two env vars:**

1. **`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`** — the TeamLead uses agent teams to add specialists as named teammates and `SendMessage` to relay human answers back to them.
2. **`CLAUDE_CODE_DISABLE_1M_CONTEXT=1`** — forces every agent to standard 200K context instead of Opus's plan-dependent 1M auto-upgrade. Cuts token burn and avoids Sonnet-teammate extra-usage errors on Max plans.

**One output style:**

3. **`outputStyle: "jfdi-agent"`** — the plugin ships a custom output style at `${CLAUDE_PLUGIN_ROOT}/output-styles/jfdi-agent.md` that strips Claude Code's coding-focused default system prompt (`# Doing tasks`, `# Committing changes with git`, `# Creating pull requests`, the trailing "do not write .md files" directive, etc.) for every agent in the team. Most agents author Markdown artefacts — visions, architecture docs, decisions logs, demo reports, minted agent definitions — not code, and the coding defaults actively conflict with their roles. The minted Developer agents (and Verifier, for throw-away diagnostics) include coding guidance explicitly in their own bodies. See `${CLAUDE_PLUGIN_ROOT}/docs/system-prompt-composition.md` for the full rationale.

Agent teams also requires Claude Code v2.1.32 or later. You cannot check the version reliably from inside a Bash tool call; just mention the requirement in the final summary.

**Use your native tools** — `Read`, `Write`, and `Edit` — to ensure all three entries are present. Do not use a Node one-liner or a Bash heredoc; those look alarming to a human watching the bootstrap for the first time. The approach:

1. **Create the directory** if it doesn't exist: `mkdir -p .claude` (a simple, one-line Bash call).
2. **Read** `.claude/settings.json` with the `Read` tool. If it doesn't exist, that's fine — you'll create it.
3. **If the file doesn't exist**, use the `Write` tool to create it with:
   ```json
   {
     "env": {
       "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
       "CLAUDE_CODE_DISABLE_1M_CONTEXT": "1"
     },
     "outputStyle": "jfdi-agent"
   }
   ```
4. **If the file already exists**, parse the JSON you read. Check each of the three entries. If any is missing or has a different value:
   - For the two env vars: add/update them in the `env` block, preserving any other `env` keys the user has set.
   - For `outputStyle`: if it is missing, set it to `"jfdi-agent"`. If it is present with a **different value** (the user has a custom preference), **do not overwrite** — print a one-line warning and leave it alone. Respecting the user's explicit choice is more important than automation.
   - Use `Write` rather than `Edit` for JSON merges because `Edit` is fragile with whitespace in JSON files.

After the write (or no-op), report one line per entry:

> Ensured `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, `CLAUDE_CODE_DISABLE_1M_CONTEXT=1`, and `outputStyle: "jfdi-agent"` in `.claude/settings.json`.

**Bootstrap never stops at Step 1b.** The settings are infrastructure the TeamLead needs later, not something bootstrap depends on right now.

### 1d. Project state survey

**Important.** Do *not* check directory existence with bare `ls -la <path>`. On a fresh repo, most of the paths this skill cares about do not exist yet, and `ls` on a missing path exits non-zero.

Use a single idempotent state survey instead:

```bash
for p in vision docs docs/demos .claude .claude/agents vision/README.md docs/README.md; do
  if [ -e "$p" ]; then echo "exists: $p"; else echo "missing: $p"; fi
done
```

Read the output and branch your subsequent steps on it:

- any directory missing → create in Step 2
- any file in Step 3 present → leave it alone

Do this survey *before* any other state-dependent action.

## Step 2: Create the shared directory structure

In the user's project, create these directories if they don't exist:

- `vision/`
- `docs/`
- `docs/demos/`
- `.claude/agents/` — where the Architect will mint per-layer developer agents during Stage 2

Use `mkdir -p` so the call is idempotent.

That is the whole directory tree. Deliberately small. No `cycles/`, no `features/`, no per-stage subdirectories. Files are added by the owning agents as the project progresses.

## Step 3: Seed placeholder READMEs

Create these files if they don't exist (use the Step 1d survey output — do not re-check with `ls`). Small stubs; the owning agents will fill the content later. If a file already exists, leave it alone and move on.

### 3a. `vision/README.md`

```markdown
# Product Vision

This directory is owned by **ProductOwner** and contains the product's reason for existing:
its intent, users, goals, non-goals, and constraints.

Expected contents (created during the intake interview):

- `overview.md` — elevator pitch, problem, users
- `goals.md` — success criteria and non-goals
- `constraints.md` — regulatory, commercial, technical constraints, including the `## Architectural components` list (runtime pieces the system is composed of) and the `## Code review platform` line (`github | gitlab | bitbucket | none`)
- `personas.md` — end-user and operator personas
- `glossary.md` — user-facing vocabulary
- `acceptance.md` — numbered one-line acceptance items (co-authored with Architect in Stage 2)

To start the workflow (including the intake interview):

    claude --agent jfdi-agents:team-lead

The TeamLead will create an agent team (after asking for your confirmation) and spawn ProductOwner as an interactive teammate to run the intake.
```

### 3b. `docs/README.md`

```markdown
# Docs

Team-owned documentation. Sized deliberately light.

| File | Owner | Purpose |
|---|---|---|
| `architecture.md` | Architect | Layers, folder map, developer roster, technology stack |
| `decisions.md` | Architect | Append-only one-line log of non-obvious technical decisions |
| `demos/<date>-<slug>.md` | Verifier | One demo report per Build milestone / Refine pass |

None of these are created by bootstrap — they are created by the owning agent in their first session.
```

### 3c. `.claude/agents/README.md`

```markdown
# Project-scoped subagents

This directory holds **Architect-minted developer agents**, one per layer. Each is a project-scoped Claude Code subagent the TeamLead spawns as a teammate during the Build and Refine stages.

The Architect writes these files during Stage 2 (Architecture & team design) using the `/jfdi-agents:write-agent` skill. Typical contents after Stage 2 for a three-layer project:

    .claude/agents/
    ├── data-dev.md
    ├── backend-dev.md
    ├── frontend-dev.md
    └── shared-dev.md   # if the Architect declared a shared layer

The human is welcome to tweak these files by hand. The `/jfdi-agents:write-agent` skill is also available to humans who want to regenerate one with different parameters.

**Hard rule.** Each developer agent owns exactly one folder. Do not edit a developer's `owned-folder` clause without also updating `docs/architecture.md`'s Folder map section — they must agree.
```

## Step 4: Ensure `.claude-state/` is git-ignored

The recommended per-project isolation pattern stores Claude Code state (settings, credentials, plugins, teams) at `./.claude-state/` inside the project. Credentials in particular must not be committed.

- **If `.gitignore` exists** and does not already match `.claude-state/`, append a section:
  ```
  # Claude Code per-project state directory (see /jfdi-agents:bootstrap output)
  .claude-state/
  ```
- **If `.gitignore` does not exist**, create one containing just that section.
- Do not touch an existing `.gitignore` that already ignores the path (match on a line starting with `.claude-state`).

This step is idempotent. If you added the line, report it in the final summary.

## Step 5: Materialise `main` if HEAD is unborn

The TeamLead's RepoSteward branches every stage off `main` — but a freshly-`git init`'d repo with no commits has no `main` (HEAD is unborn until the first commit lands). Without a starting commit, RepoSteward correctly refuses to branch ("nothing to branch from") and the workflow blocks before it gets going.

Bootstrap closes this gap by making a single initial commit when the repo doesn't yet have one.

**Detection:**

```bash
if git rev-parse --verify HEAD >/dev/null 2>&1; then
  echo "head_present"
else
  echo "head_unborn"
fi
```

**Behaviour:**

- **`head_unborn`** → make the initial commit:
  ```bash
  git add -A
  git commit -m "chore: scaffold jfdi-agents project

  Created by /jfdi-agents:bootstrap. Establishes main with the project
  scaffolding (vision/, docs/, .claude/agents/, settings, gitignore) so
  the TeamLead's RepoSteward can branch from it.

  Next: launch the TeamLead with \`claude --agent jfdi-agents:team-lead\`."
  ```
  Use the **human's normal git config** as the author — bootstrap is a skill running interactively under the human's session, not an agent. Do not pass `--author=` and do not invoke `commit-as-agent`. Report the commit's short SHA in the final summary.

- **`head_present`** → leave the working tree alone. Bootstrap is idempotent on already-initialised repos; if the human has scaffolded files that aren't yet committed, that's their decision to handle when they're ready. Do not silently `git add -A && commit` over an existing tree — the human may have other in-flight work.

**Edge case:** if `head_unborn` and `git status --porcelain` produces no output (the working tree is empty after Steps 2–4 — somehow nothing was created), skip the commit. There is nothing to commit, and an empty `--allow-empty` commit just to materialise `main` adds noise without value. Report the empty state in the final summary; the human can intervene.

## Step 6: Final summary

Report to the user:

```
jfdi-agents bootstrap complete.

Created:
  vision/
  docs/
  docs/demos/
  .claude/agents/
  vision/README.md
  docs/README.md
  .claude/agents/README.md
  .claude/settings.json (env + output style)

Git state:
  <one of:>
    - Made initial commit on main (<short-sha>): "chore: scaffold jfdi-agents project".
      RepoSteward will be able to branch from main when the TeamLead launches.
    - HEAD already had commits — left the tree alone.
      The scaffolding files above are uncommitted; please commit them
      before launching the TeamLead, or RepoSteward will see a dirty
      tree on the first branch open.
    - HEAD unborn but working tree empty after scaffolding — no commit made.
      Investigate; bootstrap should have produced files.

Per-project isolation (STRONGLY RECOMMENDED before you launch)
-----------------------------------------------
Claude Code stores all state (settings, credentials, plugins, and the
agent-teams directory) under CLAUDE_CONFIG_DIR — default ~/.claude.
Every project on your machine shares that directory by default.

This matters for the TeamLead: its zombie-recovery routine can
`rm -rf` under $CLAUDE_CONFIG_DIR/teams/<name>/. With the default
~/.claude/, a wrong-guess team name there can destroy another
project's team state.

To scope this session's state entirely to the current project:

  export CLAUDE_CONFIG_DIR="$PWD/.claude-state"
  mkdir -p "$CLAUDE_CONFIG_DIR"

Once set, every `claude ...` command in this shell uses ./.claude-state/
for everything — teams, tasks, projects, plugins, auth. Cross-project
collisions are impossible.

Caveats:
  - First launch in a new CLAUDE_CONFIG_DIR wants a fresh `claude auth
    login` and `/plugin install jfdi-agents@jfdi-agents`.
  - .claude-state/ contains credentials — add it to .gitignore.

Next step — launch the TeamLead
-----------------------------------------------
The TeamLead is the one-shell-command entry point to the whole workflow.
It runs as a main-session agent that reads the project state, figures out
which stage the team is in (starting from "no Vision yet" if this project
is fresh), and adds the appropriate specialist as a foreground teammate.
Questions from the specialist are piped back to the TeamLead session for
you to answer directly — no exit-and-relaunch between stages.

  1. Exit this session:

       /exit

  2. (Recommended) Scope state to this project, then launch:

       export CLAUDE_CONFIG_DIR="$PWD/.claude-state"
       mkdir -p "$CLAUDE_CONFIG_DIR"
       claude --agent jfdi-agents:team-lead

     (If running in dev mode with --plugin-dir, prepend the flag:
      claude --plugin-dir "<absolute-path-to-plugin-checkout>" \
             --agent jfdi-agents:team-lead)

  3. When prompted, tell the TeamLead how you want it to run. Default
     is Checkpointed mode, which pauses for your approval after each major
     stage. Say something like:

       "Proceed in Checkpointed mode. Start from zero — this is a fresh
       project. Run the Vision intake with me, then drive through
       architecture & team design, the walking-skeleton build, and
       refinement — pausing for my approval after each stage."

     Autonomous JFDI mode is also available for users who trust the team
     to run without intervention; see /jfdi-agents:start for both prompts.
     (Note: "Autonomous JFDI" is strict — no human checkpoint means every
     step runs, including the sequential-skeleton rule and every Verifier
     pass.)

The TeamLead's first action will be to ask you to confirm creation of a
Claude Code agent team. Confirm "Proceed" and it'll spawn ProductOwner
as an interactive teammate for the intake. Every question during intake
comes to you as an AskUserQuestion in the TeamLead's session — you never
have to switch between sessions. When the intake is done, the TeamLead
spawns Architect alongside ProductOwner for Stage 2, which produces the
architecture doc, the acceptance list, and one `.claude/agents/<layer>-dev.md`
per minted developer. After that, Build runs one layer at a time.

There is only one supported way to run this plugin: launching the
TeamLead as shown. Specialist main-session launches like
`claude --agent jfdi-agents:product-owner` are not supported — the
specialists cannot talk to the human on their own (AskUserQuestion is
disallowed at the frontmatter level), so they only function as
teammates of the TeamLead-led team.

Read ${CLAUDE_PLUGIN_ROOT}/docs/terminology.md and
${CLAUDE_PLUGIN_ROOT}/docs/process.md first if you want to know the
full flow. (${CLAUDE_PLUGIN_ROOT} resolves to the plugin's install
directory — NOT your home directory, NOT your project's .claude — it's
wherever the jfdi-agents plugin is installed on your machine.)
```

## Hard rules

- **Idempotent.** If a file exists, do not overwrite it. Check before creating.
- **No surprises.** Report every file the skill created. No silent changes.
- **Do not run intake.** The bootstrap ends with a pointer to the Vision intake, it does not run it. The intake is the ProductOwner's job in its own session.
- **Do not install any frameworks, bundlers, test runners, or runtimes.** Those are Architect decisions, taken during Stage 2, installed by the developer agents during Build.
- **The initial commit (Step 5) only fires when HEAD is unborn.** If commits already exist, the human owns when and how to commit the scaffolding — bootstrap does not silently `git add -A && commit` over an existing tree. The commit's author is the human's git config (no `--author=` slug), because the bootstrap skill runs interactively under the human's session, not as an agent.
