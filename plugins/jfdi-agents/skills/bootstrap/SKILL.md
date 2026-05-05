---
name: bootstrap
description: One-stop pre-flight for starting a new project with the jfdi-agents plugin. Run this as the first thing after installing the plugin. Checks prerequisites (git), initialises a git repo if needed, writes a comprehensive `.claude/settings.json` (default agent, output style, agent-teams env vars, bypassPermissions mode, sandboxing, marketplace pre-registration, plugin auto-enable), creates the shared directory structure (`vision/`, `docs/`, `docs/demos/`, `.claude/agents/`), seeds placeholder READMEs, and makes an initial commit if the repo is empty. Idempotent — safe to re-run; existing settings are merged not overwritten. Use this skill whenever a user installs the plugin for the first time, asks "how do I set up jfdi-agents in this repo?", or needs to repair a partially-bootstrapped project. After this completes, the project is ready for plain `claude` — the settings.json drives the rest.
---

You are bootstrapping a new project for the `jfdi-agents` team. This skill is the human's one-stop prep command — it gets everything in order so the TeamLead can launch and run without bouncing the human back out for environment fixes. Idempotent — if something is already in place, leave it alone. If something is missing, create it. Never overwrite existing files unless you are explicitly merging a known JSON structure (like `.claude/settings.json`).

## Context about the plugin

This plugin ships a six-role agent team (see `agents/` inside the plugin install directory) that drives software development using the JFDI pattern: Product Owner + Architect design a layered system, Architect mints per-layer developer agents, the team walks the stack sequentially for a walking skeleton, then parallelises refinement. The team operates on a specific directory structure in the *user's* project, which this skill creates.

The specific languages, frameworks, and the per-layer developer agents are **not** chosen at bootstrap. The Architect makes those decisions during Stage 2 (Architecture & team design), once the Vision is known. Bootstrap only lays down empty structure.

## Step 1: Check prerequisites

### 1a. Git repository

The plugin's whole workflow depends on git — feature branches per stage, Verifier sign-off via demo reports, incremental commits. If the current directory is not already a git repo, initialise one:

```bash
git rev-parse --is-inside-work-tree 2>/dev/null && echo "git-ready" || git init
```

If `git init` fails (e.g. no `git` on `PATH`), stop and tell the user:

> `git init` failed — git may not be installed or available in this shell. Please install git and re-run bootstrap.

### 1b. Claude Code project settings — comprehensive `settings.json`

Bootstrap writes a **complete** `.claude/settings.json` that drives every aspect of how `claude` behaves in this project. The goal: the human can `cd /path/to/project` and run plain `claude` — no flags, no env-var exports, no shell wrappers. The settings file does the work.

**The full target shape:**

```json
{
  "agent": "jfdi-agents:team-lead",
  "outputStyle": "jfdi-agent",
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1",
    "CLAUDE_CODE_DISABLE_1M_CONTEXT": "1"
  },
  "permissions": {
    "defaultMode": "bypassPermissions",
    "skipDangerousModePermissionPrompt": true
  },
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "filesystem": {
      "allowRead": [".", "~/.claude"],
      "allowWrite": [".", "~/.claude"],
      "denyRead": [
        "~/.aws/**",
        "~/.ssh/**",
        "~/.gnupg/**",
        "~/.netrc",
        "~/.bash_history",
        "~/.zsh_history",
        "~/.fish_history",
        "~/.config/git/credentials"
      ],
      "denyWrite": [
        "~/.aws/**",
        "~/.ssh/**",
        "~/.gnupg/**"
      ]
    }
  },
  "extraKnownMarketplaces": {
    "jfdi-agents": {
      "source": {
        "source": "github",
        "repo": "dr-zog/jfdi-agents"
      }
    }
  },
  "enabledPlugins": {
    "jfdi-agents@jfdi-agents": true
  }
}
```

**What each block does:**

- **`agent`** — sets the default main-thread agent. Plain `claude` in this directory now launches as `jfdi-agents:team-lead` with no `--agent` flag.
- **`outputStyle`** — strips Claude Code's coding-focused default prompt for every agent. Load-bearing for system-prompt composition; see `${CLAUDE_PLUGIN_ROOT}/docs/system-prompt-composition.md`.
- **`env`** — `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` enables the agent-teams machinery the TeamLead depends on. `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` forces standard 200K context to cut token burn and avoid Sonnet-teammate extra-usage errors on Max plans.
- **`permissions.defaultMode: "bypassPermissions"`** — auto-approves tool calls. Equivalent to `--dangerously-skip-permissions`. Combined with the sandbox below, this is safer than it sounds: the sandbox restricts what tool calls can actually do at the OS level, while bypassPermissions just removes the prompts.
- **`permissions.skipDangerousModePermissionPrompt`** — skips the warning prompt about bypass mode. The user has already opted in via this settings file.
- **`sandbox`** — OS-level filesystem isolation. Reads and writes are restricted to the project directory and `~/.claude/` (Claude's own state). Common credential paths (`~/.aws/`, `~/.ssh/`, `~/.gnupg/`, history files, `~/.netrc`) are explicitly denied. This is the structural defence against the failure mode where Claude tries to be helpful by reading the user's home directory.
- **`extraKnownMarketplaces.jfdi-agents`** — pre-registers the marketplace so first-launch in a fresh project doesn't silently fall back to the default Claude prompt because it never knew about us.
- **`enabledPlugins["jfdi-agents@jfdi-agents"]: true`** — auto-enables the plugin alongside the marketplace registration. First launch installs and loads the plugin without needing `/plugin install` manually.

Agent teams requires Claude Code **v2.1.32 or later**. You cannot check the version reliably from inside a Bash tool call; mention the requirement in the final summary.

**Use your native tools** — `Read`, `Write`, and `Edit` — to ensure all entries are present. Do not use a Node one-liner or a Bash heredoc; those look alarming to a human watching the bootstrap for the first time. The approach:

1. **Create the directory** if it doesn't exist: `mkdir -p .claude` (a simple, one-line Bash call).
2. **Read** `.claude/settings.json` with the `Read` tool. If it doesn't exist, that's fine — you'll create it.
3. **If the file doesn't exist**, use the `Write` tool to create it with the full target shape above.
4. **If the file already exists**, parse the JSON you read. **Merge** rather than overwrite, with this policy per top-level key:

   | Key | If missing | If present and matches | If present and differs |
   |---|---|---|---|
   | `agent` | add | leave | warn, leave (respect user's pin) |
   | `outputStyle` | add | leave | warn, leave |
   | `env` | add | merge keys (preserve user's other env vars) | per-key: add missing, warn on differences |
   | `permissions.defaultMode` | add | leave | warn, leave |
   | `permissions.skipDangerousModePermissionPrompt` | add | leave | warn, leave |
   | `permissions` (other keys) | leave entirely | — | — |
   | `sandbox` | add (whole block) | leave (sandbox configs are intricate; don't try to deep-merge) | leave |
   | `extraKnownMarketplaces.jfdi-agents` | add | leave | warn, leave |
   | `extraKnownMarketplaces` (other entries) | leave | — | — |
   | `enabledPlugins["jfdi-agents@jfdi-agents"]` | add (true) | leave | warn (e.g. user set false), leave |

   Use `Write` rather than `Edit` for JSON merges because `Edit` is fragile with whitespace in JSON files.

5. **After the write (or no-op)**, report what was added or warned about, one line per finding:

   > Ensured `agent`, `outputStyle`, `env`, `permissions`, `sandbox`, `extraKnownMarketplaces`, `enabledPlugins` in `.claude/settings.json`.
   >  _(For each warning: "WARN: existing `agent` differs from `jfdi-agents:team-lead` — left alone")_

**Bootstrap never stops here.** The settings drive the launch experience but bootstrap still needs to scaffold the directory tree.

### 1c. Project state survey

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

Create these files if they don't exist (use the Step 1c survey output — do not re-check with `ls`). Small stubs; the owning agents will fill the content later. If a file already exists, leave it alone and move on.

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
  .claude/settings.json — full project config (agent, output style, env,
                          permissions, sandbox, marketplace, plugin enable)

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

How to launch
-----------------------------------------------
Exit this session and run plain `claude` in this directory:

  /exit

  cd /path/to/this/project   (if you're not already there)
  claude

The settings.json bootstrap just wrote tells Claude Code to:
  - launch as the TeamLead (`agent: "jfdi-agents:team-lead"`)
  - register the JFDI marketplace and auto-enable the plugin
  - apply the JFDI output style across every agent in the session
  - run with bypass-permissions mode (no per-tool prompts)
  - sandbox tool calls to this project + ~/.claude (no leakage to your
    home directory; ~/.aws, ~/.ssh, ~/.gnupg, history files, .netrc all
    explicitly denied)
  - export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 and the 1M-context
    disable into every Claude session in this dir

For continuation across sessions:

  claude -c

That's it. No `--agent` flag, no `--dangerously-skip-permissions`,
no env-var exports.

First time here?
-----------------------------------------------
The first time you run `claude` in a directory bootstrap-ed for
jfdi-agents, Claude Code will prompt you to register the marketplace
and install the plugin (one-time consent — the settings.json declared
both, but a human still confirms the trust dance). Approve those, and
every subsequent launch in this directory is silent.

How to brief the TeamLead
-----------------------------------------------
The TeamLead's first action is to ask you to confirm creation of a
Claude Code agent team. Confirm "Proceed". Then prompt it with how
you want it to run. Default is Checkpointed mode (pauses for approval
after each major stage):

  "Proceed in Checkpointed mode. Start from zero — this is a fresh
  project. Run the Vision intake with me, then drive through
  architecture & team design, the walking-skeleton build, and
  refinement — pausing for my approval after each stage."

Autonomous JFDI mode is available for users who trust the team to
run without intervention; see /jfdi-agents:start for both prompts.
"Autonomous JFDI" is strict — no human checkpoint means every step
runs, including the sequential-skeleton rule and every Verifier pass.

There is only one supported way to run this plugin: launching as the
TeamLead (which the settings.json does for you). Specialist
main-session launches like `claude --agent jfdi-agents:product-owner`
are not supported — the specialists cannot talk to the human on their
own (AskUserQuestion is disallowed at the frontmatter level), so they
only function as teammates of the TeamLead-led team.

Optional: per-project isolation via CLAUDE_CONFIG_DIR
-----------------------------------------------
The settings.json above is sufficient for safe operation — sandbox
prevents filesystem leakage and the team-name verification in the
TeamLead body prevents cross-project zombie-recovery damage.

If you want even stronger isolation (per-project teams directory,
per-project plugin cache, per-project credentials), set:

  export CLAUDE_CONFIG_DIR="$PWD/.claude-state"
  claude

This is optional. The cost is per-project `claude auth login`. Not
recommended unless you're running multiple high-stakes projects on
the same machine and want belt-and-suspenders. `.claude-state/` is
already in .gitignore so credentials won't be committed.

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
