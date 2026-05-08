# Installing the plugin

> **TL;DR.** Two slash-commands install the plugin from the public marketplace, then `/jfdi-agents:bootstrap` prepares the project (writing `.claude/settings.json` and a `./jfdi.sh` launcher), then `./jfdi.sh` becomes the only command you ever need to run from then on.

## The three modes Claude Code can load a plugin from

1. **An installed scope** — `~/.claude/settings.json` (user), `<project>/.claude/settings.json` (project), or `<project>/.claude/settings.local.json` (local). A plugin registered here is loaded by every new Claude Code session in that scope.
2. **A marketplace** — a catalog file at `.claude-plugin/marketplace.json` somewhere Claude Code can see it. Once you `/plugin marketplace add <source>` it, you can `/plugin install <plugin>@<marketplace>` to move the plugin into one of the installed scopes above.
3. **The `--plugin-dir` CLI flag** — a one-off, session-scoped load. Only that one session sees the plugin. The next `claude` invocation has no memory of it.

For end-users running the plugin to build a product, mode 2 is correct — install once, never think about it again. For maintainers iterating on the plugin's source, mode 3 (dev mode) gives a fast reload loop without the install-cache dance.

## End-user install — from the public `dr-zog/ai-marketplace` marketplace

The plugin is published to the [`dr-zog/ai-marketplace`](https://github.com/dr-zog/ai-marketplace) marketplace on GitHub. That marketplace's `marketplace.json` declares `jfdi-agents` (alongside any other plugins the marketplace publishes) and points at the [`dr-zog/jfdi-agents`](https://github.com/dr-zog/jfdi-agents) source repo.

**The marketplace tracks `main`, no version pinning.** The `jfdi-agents` entry in `dr-zog/ai-marketplace`'s catalog uses a `git-subdir` source pointing at `dr-zog/jfdi-agents` with **no `ref` field**, so Claude Code resolves it against `HEAD` of the default branch (`main`). Both the initial `/plugin install` and every subsequent `/plugin update` always pull the latest published release. There is no specific version to opt in to; the latest is whatever's been most recently published by the GitLab `publish-to-github` CI stage. If you need to roll back to a specific past version, you have to `--plugin-dir` against a local checkout pinned to that tag — there is no version selector in the marketplace install path.

**Step 1. Add the marketplace.**

From any running Claude Code session:

```
/plugin marketplace add dr-zog/ai-marketplace
```

Claude Code clones the marketplace repo, reads its `marketplace.json`, and registers the marketplace under the name declared there (`dr-zog`). The registration persists in `<CLAUDE_CONFIG_DIR>/plugins/known_marketplaces.json` — you only do this once per machine *per Claude home*. (See "Per-project Claude home" below — under the `./jfdi.sh` launcher, each JFDI project has its own marketplace registration, which is the correct isolation.)

**Step 2. Install the plugin.**

Still in a Claude Code session:

```
/plugin install jfdi-agents@dr-zog
```

The `jfdi-agents@dr-zog` form is `<plugin-name>@<marketplace-name>`. Claude Code fetches the plugin from `dr-zog/jfdi-agents`, copies it into the versioned cache at `<CLAUDE_CONFIG_DIR>/plugins/cache/dr-zog-jfdi-agents/<version>/`, and writes an entry to `<CLAUDE_CONFIG_DIR>/settings.json` (or wherever you scoped it) under `enabledPlugins`.

Default scope is **user** — the plugin loads for every Claude Code session you run in that Claude home. Pass `-s project` or `-s local` to the CLI form if you want a different scope:

```bash
claude plugin install jfdi-agents@dr-zog --scope project
```

**Step 3. Run the bootstrap pre-flight.**

```
/jfdi-agents:bootstrap
```

This is the plugin's pre-flight skill. It writes a comprehensive `.claude/settings.json`, lays down the directory tree (`vision/`, `docs/`, `docs/demos/`, `.claude/agents/`, `.claude-state/`), and generates a `./jfdi.sh` launcher script that pins `CLAUDE_CONFIG_DIR=$PWD/.claude-state` so this JFDI project has its own isolated Claude home. The settings.json bootstrap writes also pre-registers the marketplace and auto-enables the plugin, so the per-project Claude home picks up the plugin on first launch.

**Step 4. Verify and launch.**

```
/exit
```

Then from your terminal, in the project directory:

```bash
./jfdi.sh
```

`./jfdi.sh` exports `CLAUDE_CONFIG_DIR=$PWD/.claude-state` and execs `claude`. Because this is a fresh per-project Claude home, you'll see the marketplace-registration prompt and the plugin-install prompt once on first launch — approve them. After that, every subsequent `./jfdi.sh` is silent.

To continue an existing session: `./jfdi.sh -c` (the launcher passes args through to `claude`).

### Updating

The marketplace tracks `main` of the published mirror without a `ref` pin (see the note at the top of this section), so updates always pull the latest release. The two-step is:

```
/plugin marketplace update dr-zog
```

(Updates the cached marketplace catalog.)

```
/plugin update jfdi-agents@dr-zog
```

(Updates the installed plugin to whatever the catalog now resolves to — i.e. latest published.)

Claude Code fetches the new version, copies it into a new versioned cache directory at `<CLAUDE_CONFIG_DIR>/plugins/cache/dr-zog-jfdi-agents/<version>/`, and cleans up the old one after seven days. There is no "stay on v0.7.x" mode; if you need version stability for a project, use dev mode (`--plugin-dir` against a checkout you control) instead of the marketplace install.

### Uninstalling

```
/plugin uninstall jfdi-agents@dr-zog
/plugin marketplace remove dr-zog
```

### Per-project Claude home

The `./jfdi.sh` launcher generated by bootstrap exports `CLAUDE_CONFIG_DIR=$PWD/.claude-state`. The consequence:

- Each JFDI project has its own marketplace registration, plugin cache, credentials, teams directory, and tasks directory — all under `./.claude-state/`.
- A stalled JFDI team in one project cannot collide with another project's state.
- Your personal `~/.claude/` (the Claude home you use for non-JFDI work) is left untouched and is **explicitly denied** at the OS-sandbox level by the bootstrap-generated settings.json — JFDI sessions cannot read or write into it.

The cost: the first launch in each JFDI project asks for plugin install (one-time consent) and `claude auth login` (per-project credentials, since the per-project state dir starts empty). Both are silent on subsequent launches.

`.claude-state/` is added to the project's `.gitignore` by bootstrap, so credentials and runtime state never reach the repo.

### Auto-prompt on project open (optional)

Bootstrap-generated `.claude/settings.json` already does this — it pre-registers the marketplace under `extraKnownMarketplaces.dr-zog` and pre-enables the plugin under `enabledPlugins["jfdi-agents@dr-zog"]`. Anyone who clones a JFDI-bootstrap-ed project, trusts the folder, and launches via `./jfdi.sh` will be prompted to register the marketplace and install the plugin on first run, then never again.

## Maintainer install — `--plugin-dir` for active plugin development

The install path above is for using the plugin. If you are actively *editing* the plugin itself — tweaking agent bodies, changing skills, iterating on the shared docs — install mode is the wrong fit. Every time you change a file you would have to bump the version, push, wait for the marketplace catalog to update, and `/plugin update` to pull it into the cache.

For active development, stay in dev mode:

```bash
claude --plugin-dir "/absolute/path/to/jfdi-agents-checkout"
```

Changes to files in the checkout are picked up on `/reload-plugins` without any install-step rebuild. A shell alias removes the friction:

```bash
# ~/.bashrc or ~/.zshrc
jfdi-dev() {
  claude --plugin-dir "/home/you/code/jfdi-agents" "$@"
}
```

Then:

```bash
jfdi-dev --agent jfdi-agents:team-lead
```

(Note: dev-mode invocations are the one place `claude --agent` is still appropriate, because dev mode bypasses the project-level settings.json that would otherwise pick the agent.)

Dev mode and installed mode can coexist. If an installed copy of the plugin is already registered and you also pass `--plugin-dir` pointing at a checkout of the same name, the local `--plugin-dir` copy takes precedence for that one session. This lets you test an in-progress change against a stable install without uninstalling first.

## Relationship to `.claude/settings.json`

Installation scope is separate from project settings. Your project's `.claude/settings.json` continues to hold project-level settings (default agent, output style, env vars, sandbox, marketplace registration, plugin enable); the plugin installation happens in `<CLAUDE_CONFIG_DIR>/settings.json` (user scope, default) or wherever you specified with `--scope`. The two do not collide.

Under the `./jfdi.sh` launcher, `<CLAUDE_CONFIG_DIR>` is `./.claude-state/` — so the project-scoped `.claude/settings.json` and the per-project user-scoped `./.claude-state/settings.json` both live inside the project, but they serve different purposes. Bootstrap only writes the former; the latter is populated by Claude Code itself when you install the plugin.

## Which mode should you be in?

| Situation | Mode |
|---|---|
| Using the plugin to build a product | **Marketplace install + bootstrap + `./jfdi.sh`**. One-time setup; every future session is flagless. |
| Editing the plugin's own agent definitions, skills, or docs | **Dev mode** (`--plugin-dir` + shell alias). Fast reload loop, no install dance. |
| Both at once | Install it *and* use `--plugin-dir` when you're hacking — the dev copy wins for the session that passes the flag, the installed copy wins everywhere else. |

For the typical `jfdi-agents` end-user — the person who installs it, defines a product vision, and runs the TeamLead — the four steps in the end-user section above are the whole story. After that, never think about `--plugin-dir` again.
