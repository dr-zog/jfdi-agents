# Running the plugin without `--plugin-dir`

> **TL;DR.** Every `claude --agent jfdi-agents:<name>` command in this plugin's docs assumes the plugin is installed to a persistent Claude Code scope. If you cloned the repo and ran `claude --plugin-dir ./jfdi-agents`, the plugin is **not** installed — it's loaded for that one session only. This doc tells you how to install it properly, once, so every future session picks it up automatically.

## The problem

Claude Code loads plugins from three places:

1. **An installed scope** — `~/.claude/settings.json` (user), `<project>/.claude/settings.json` (project), or `<project>/.claude/settings.local.json` (local). A plugin registered here is loaded by every new Claude Code session in that scope.
2. **A marketplace** — a catalog file at `.claude-plugin/marketplace.json` somewhere Claude Code can see it. Once you `/plugin marketplace add <source>` it, you can `/plugin install <plugin>@<marketplace>` to move the plugin into one of the installed scopes above.
3. **The `--plugin-dir` CLI flag** — a one-off, session-scoped load. Only that one session sees the plugin. The next `claude` invocation has no memory of it.

If you're in mode 3, every `claude --agent jfdi-agents:<name>` you type without `--plugin-dir` fails. This is deliberate — `--plugin-dir` is for iterating on a plugin, not for running one.

## Recommended — install from the plugin repo's git URL

The plugin repo ships its own `.claude-plugin/marketplace.json` at the root, which means the repo itself *is* a valid marketplace containing exactly one plugin: `jfdi-agents`. You can add the repo URL directly as a marketplace and install from it in two commands.

**Step 1. Add the marketplace.**

From any running Claude Code session:

```
/plugin marketplace add https://github.com/dr-zog/jfdi-agents.git
```

Substitute your own remote URL if you've forked the plugin or are hosting it elsewhere. Any git URL works — GitHub, GitLab (self-hosted or cloud), Bitbucket, SSH, HTTPS, with or without `.git`.

Claude Code clones the repo, reads `.claude-plugin/marketplace.json`, and registers the marketplace under the name declared there (`jfdi-agents`). The registration persists in `~/.claude/plugins/known_marketplaces.json` — you only do this once per machine.

**Step 2. Install the plugin.**

Still in a Claude Code session:

```
/plugin install jfdi-agents@jfdi-agents
```

The `jfdi-agents@jfdi-agents` form is `<plugin-name>@<marketplace-name>` — they are the same name because the marketplace contains exactly one plugin, itself. Claude Code fetches the plugin from the repo, copies it into the versioned cache at `~/.claude/plugins/cache/jfdi-agents-jfdi-agents/<version>/`, and writes an entry to `~/.claude/settings.json` (or wherever you scoped it) under `enabledPlugins`.

Default scope is **user** — the plugin loads for every Claude Code session you run, in every project. Pass `-s project` or `-s local` to the CLI form if you want a different scope:

```bash
claude plugin install jfdi-agents@jfdi-agents --scope project
```

**Step 3. Verify.**

```
/plugin list
```

You should see `jfdi-agents@jfdi-agents` marked enabled. From here on, every new Claude Code session loads the plugin automatically — no `--plugin-dir`, no flags.

In a project where `/jfdi-agents:bootstrap` has run, the project's `.claude/settings.json` declares `jfdi-agents:team-lead` as the default agent, so plain `claude` Just Works:

```bash
cd /path/to/your/jfdi-project
claude
```

If you want to override (e.g. launch the TeamLead from a directory that hasn't been bootstrap-ed for jfdi-agents), the explicit form still works:

```bash
claude --agent jfdi-agents:team-lead
```

### Updating

When the plugin repo gets new commits, pull them into your installed copy:

```
/plugin marketplace update jfdi-agents
```

(Updates the cached marketplace catalog.)

```
/plugin update jfdi-agents@jfdi-agents
```

(Updates the installed plugin.)

Claude Code fetches the new commits, copies them into a new versioned cache directory, and cleans up the old one after seven days.

### Uninstalling

```
/plugin uninstall jfdi-agents@jfdi-agents
/plugin marketplace remove jfdi-agents
```

### Private repo authentication

If you're installing from a **private** repo (e.g. a self-hosted GitLab fork), the marketplace URL relies on your existing git credential setup. For an SSH URL like `git@gitlab.example.com:<your-group>/jfdi-agents.git`, if `git clone <url>` works in your shell, `/plugin marketplace add <url>` works in Claude Code — both use the same credential helpers. For the background auto-update that Claude Code runs at session start (which cannot prompt for credentials), you may need to set `GITLAB_TOKEN` (or `GITHUB_TOKEN`, `BITBUCKET_TOKEN`, etc.) in your shell environment. See [the Claude Code docs on private repositories](https://code.claude.com/docs/en/plugin-marketplaces#private-repositories) for the full list.

### Project-scoped auto-enable (optional)

If you want a downstream project to automatically prompt for the plugin when opened in Claude Code, add this to the project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "jfdi-agents": {
      "source": {
        "source": "url",
        "url": "https://github.com/dr-zog/jfdi-agents.git"
      }
    }
  },
  "enabledPlugins": {
    "jfdi-agents@jfdi-agents": true
  }
}
```

Anyone who opens the project and trusts the folder will be prompted to register the marketplace and install the plugin on first run. Useful when a team project depends on the jfdi-agents workflow and you want all collaborators on the same version.

## Alternative — `--plugin-dir` for active plugin development

The install path above is for using the plugin. If you are actively *editing* the plugin itself — tweaking agent bodies, changing skills, iterating on the shared docs — install mode is the wrong fit. Every time you change a file you would have to `/plugin update` to pull it into the cache, and the cache version can drift from the source.

For active development, stay in dev mode:

```bash
claude --plugin-dir "/absolute/path/to/jfdi-agents-checkout"
```

Changes to files in the checkout are picked up on `/reload-plugins` without any install-step rebuild. A shell alias removes the friction:

```bash
# ~/.bashrc or ~/.zshrc
jfdi() {
  claude --plugin-dir "/home/you/code/jfdi-agents" "$@"
}
```

Then:

```bash
jfdi --agent jfdi-agents:team-lead
```

Dev mode and installed mode can coexist. If an installed copy of the plugin is already registered and you also pass `--plugin-dir` pointing at a checkout of the same name, the local `--plugin-dir` copy takes precedence for that one session. This lets you test an in-progress change against a stable install without uninstalling first.

## Relationship to `.claude/settings.json`

Installation scope is separate from project settings. Your project's `.claude/settings.json` continues to hold project-level settings like `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`; the plugin installation happens in `~/.claude/settings.json` (user scope, default) or wherever you specified with `--scope`. The two do not collide.

## Which mode should you be in?

| Situation | Mode |
|---|---|
| Using the plugin to build a product | **Installed** (the git URL path above). One-time setup; every future session is flagless. |
| Editing the plugin's own agent definitions, skills, or docs | **Dev mode** (`--plugin-dir` + shell alias). Fast reload loop, no install dance. |
| Both at once | Install it *and* use `--plugin-dir` when you're hacking — the dev copy wins for the session that passes the flag, the installed copy wins everywhere else. |

For the typical `jfdi-agents` end-user — the person who installs it, defines a product vision, and runs the TeamLead — the two commands in the recommended section above are the whole story. After that, never think about `--plugin-dir` again.
