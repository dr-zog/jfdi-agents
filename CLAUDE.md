# Instructions for Claude Code working in this repo

This repository is the **plugin source** for `jfdi-agents`, published via CI to [`github.com/dr-zog/jfdi-agents`](https://github.com/dr-zog/jfdi-agents) and catalogued at the [`dr-zog/ai-marketplace`](https://github.com/dr-zog/ai-marketplace) marketplace. The plugin lives at `plugins/jfdi-agents/` and contains a TeamLead conductor plus four specialists (ProductOwner, Architect, Verifier, RepoSteward) that drive software development using the **JFDI** pattern: the Architect mints per-layer developer agents into the downstream project at architecture time, the team walks the stack sequentially to build a walking skeleton, then parallelises refinement.

When you are working in *this* repo (editing the plugin), these rules apply. When you are running the plugin *against* a downstream project, a completely different set of agent system prompts applies — those are the `agents/*.md` files inside the plugin, not this file.

## Canonical references

Before editing anything non-trivial inside the plugin, read or re-read:

1. **[`plugins/jfdi-agents/docs/terminology.md`](plugins/jfdi-agents/docs/terminology.md)** — the shared vocabulary every agent uses. Never invent a parallel term.
2. **[`plugins/jfdi-agents/docs/roster.md`](plugins/jfdi-agents/docs/roster.md)** — the six roles, what they own, what they must not touch.
3. **[`plugins/jfdi-agents/docs/process.md`](plugins/jfdi-agents/docs/process.md)** — collaboration rules, conflict resolution, commit cadence, artefact templates. Cross-cutting rules every agent needs.
4. **[`plugins/jfdi-agents/docs/team-lead-playbook.md`](plugins/jfdi-agents/docs/team-lead-playbook.md)** — team-management mechanics (spawn order, stall detection, status-block format). Lead-specific satellite of process.md.
5. **[`plugins/jfdi-agents/docs/solo-agents.md`](plugins/jfdi-agents/docs/solo-agents.md)** — solo-agent satellite of process.md. When solo mode is licensed, the human-gating protocol, integration discipline, the two-licensed-paths principle.
6. **[`plugins/jfdi-agents/docs/system-prompt-composition.md`](plugins/jfdi-agents/docs/system-prompt-composition.md)** — why the plugin ships a custom output style and how agent bodies compose under it.
7. **[`plugins/jfdi-agents/docs/local-install.md`](plugins/jfdi-agents/docs/local-install.md)** — the install flow (git URL marketplace add, dev mode, the difference between the two).

If any edit you are about to make would contradict one of these, stop and surface the conflict rather than quietly drifting.

## Agent machinery: subagents + agent-teams

This plugin uses Claude Code's **subagent** mechanism *in combination with* the **agent-teams** feature — neither alone. Subagents define the specialist prompts and tool frontmatter; agent-teams lets the TeamLead spawn multiple specialists that share a task list and DM each other.

Primary references (read before editing agent frontmatter or team-coordination code):

- [Subagents](https://code.claude.com/docs/en/sub-agents) — frontmatter fields, `tools`, `disallowed-tools`, invocation model.
- [Agent teams](https://code.claude.com/docs/en/agent-teams) — teammate spawn, shared task list, DM protocol, permission inheritance.
- [Subagent vs agent team](https://code.claude.com/docs/en/features-overview#subagent-vs-agent-team) — when each is appropriate.

Key tool-resolution rules (see `plugins/jfdi-agents/docs/team-lead-playbook.md` § 4 for the full list):

- Teammate frontmatter `tools`/`disallowed-tools` is honoured. `disallowed-tools` is applied first; `tools` is resolved against the remaining pool.
- **Teammates inherit the lead's permission restrictions.** Keep the TeamLead's denylist empty; push role-specific restrictions to the specialists themselves.
- **`AskUserQuestion` is harness-blocked for every teammate**, regardless of frontmatter. Any agent whose role requires structured multi-choice questions must run as the main session. That is why TeamLead is the only supported main-session entry point; Vision intake is TeamLead-driven.
- Plugin-shipped agents load from `<CLAUDE_CONFIG_DIR>/plugins/cache/dr-zog-jfdi-agents/<version>/` — under the bootstrap-generated `./jfdi.sh` launcher this resolves to `./.claude-state/plugins/cache/dr-zog-jfdi-agents/<version>/`, not the source tree. Frontmatter edits take effect on live installs only after a version bump + `/plugin update` (or a dev-mode install).
- **The Architect-minted developer agents are project-scoped** — they live in `.claude/agents/` in the *downstream* project, not in the plugin. Edits to them take effect immediately on the next spawn; there is no plugin cache for them.

System-prompt composition:

- **The plugin ships a custom output style** at `plugins/jfdi-agents/output-styles/jfdi-agent.md` with `keep-coding-instructions: false`. The bootstrap skill writes `"outputStyle": "jfdi-agent"` into `.claude/settings.json` when preparing a project.
- **The output style applies to every agent** — main-session TeamLead, plugin-shipped teammates, and Architect-minted project-scoped developers alike. It strips the coding-focused default system prompt while retaining Claude Code's agent-harness behaviour.
- **Agent bodies start from a clean role-orientation slate.** No counter-prime sections needed. The Architect-minted Developers (who actually write production code) and Verifier (who occasionally writes throw-away diagnostic code) include coding guidance explicitly in their own bodies as positive prose.
- **CLAUDE.md reaches every agent via `<system-reminder>` delivery regardless of output-style setting.** Project-wide invariants (branching, commit authorship, etc.) belong here.

Skills for authoring new agents against these rules:

- `plugins/jfdi-agents/skills/write-team-lead/SKILL.md` — for lead agents (TeamLead-style).
- `plugins/jfdi-agents/skills/write-agent/SKILL.md` — for all non-lead agents. Used by plugin maintainers for the four shipped specialists *and* by the Architect at Stage 2 when minting per-layer developer agents into the downstream project's `.claude/agents/`.

## What lives where

### Repo-level shell (repo root)

- **`README.md`** — the user-facing top-level README. Describes the plugin and points at the install flow (marketplace add → install → bootstrap → `./jfdi.sh`). Keep the install instructions at the top.
- **`LICENSE`** — MIT, covers the whole repo.
- **`NOTICE.md`** — third-party attributions.
- **`.gitignore`** — repo-level ignores.
- **`CLAUDE.md`** — this file.

> **Note on marketplace migration (v0.6.x → v0.7.x).** Previously this repo also held a `.claude-plugin/marketplace.json` catalog so the repo URL could be added directly as a marketplace. That file has been removed; the marketplace catalog now lives in the separate [`dr-zog/ai-marketplace`](https://github.com/dr-zog/ai-marketplace) repo, alongside other plugins. The `publish-to-github` CI stage continues to publish plugin source to `dr-zog/jfdi-agents`, which is what the `dr-zog/ai-marketplace` catalog references.

### Plugin (`plugins/jfdi-agents/`)

- **`.claude-plugin/plugin.json`** — plugin manifest (name, version, description, keywords).
- **`agents/*.md`** — five plugin-shipped specialist definitions (team-lead, product-owner, architect, verifier, repo-steward). The Developer *role* is minted per-project by the Architect, not shipped here. Plugin-shipped agents cannot use `hooks`, `mcpServers`, or `permissionMode`.
- **`skills/*/SKILL.md`** — slash-command skills, namespaced `/jfdi-agents:<skill>`.
- **`output-styles/jfdi-agent.md`** — the custom output style that strips Claude Code's coding default for the whole team.
- **`scripts/stall-detector.sh`** — a small filesystem watcher the TeamLead runs in the background to detect stalled teammates.
- **`docs/*.md`** — shared references. These are the single source of truth for team behaviour; if an agent's system prompt contradicts a doc, the doc wins. Agent bodies reference them via `${CLAUDE_PLUGIN_ROOT}/docs/<file>.md` so the paths resolve correctly regardless of install mode.

## What does NOT live here

- Product code for the *thing being built by the plugin*. That lives in the downstream user's repo.
- User-project artefacts (`vision/`, `docs/demos/`, `.claude/agents/<layer>-dev.md`). Those exist in the repo that installs the plugin, not here.

## Tone for written material in this repo

- Written material should be declarative, spare, and unambiguous — these docs are read by both humans and other Claude sessions, and rhetorical flourish translates badly.
- Use the canonical vocabulary: Vision, Architecture, Layer, Acceptance list, Walking skeleton, Build, Refine, Demo. Never "sprint", "epic", "story", "ticket", "cycle", "feature" (as a planning unit), "scenario", "step" — the last four were part of the retired ATDD variant of this plugin and are now banned.
- When you must coin a new term, propose it in `plugins/jfdi-agents/docs/terminology.md` in the same commit.

## When editing an agent definition

1. Re-read the agent's entry in `plugins/jfdi-agents/docs/roster.md` first. If your edit implies a change to its scope, update the roster in the same commit.
2. The agent's body (system prompt) should reference the shared docs rather than restating them. Restating drifts; references don't. Use the `${CLAUDE_PLUGIN_ROOT}/docs/<file>.md` form so the paths resolve at runtime to the installed plugin location.
3. Any change to decision-making priors (especially Architect's tiebreaker rules, since Architect is the sole technical authority with no Mediator to fall back on) must be called out in the commit message.

## When editing shared docs

Shared docs are load-bearing. Changes propagate to every agent. Treat them the way you would treat a public API:

- Additive changes (new terms, new rules) are low-risk.
- Renames and redefinitions are breaking changes — check that no agent definition relies on the old wording before shipping.
- Removals require explicit reasoning in the commit message.

## When editing the install surface

- Changes to install commands (marketplace name, plugin slug) ripple through `plugins/jfdi-agents/skills/bootstrap/SKILL.md` (the settings.json template), the top-level `README.md` Quick Start, `plugins/jfdi-agents/docs/local-install.md`, `plugins/jfdi-agents/docs/process.md` Step 0, and `evals/README.md`. Update them all in the same MR or installs will drift.
- Changes to the launcher script `./jfdi.sh` (generated by bootstrap) ripple through every doc that mentions `./jfdi.sh` as the launch command — see the same surfaces above.
- Marketplace migrations (e.g. moving from one marketplace to another) are install-breaking for existing installs. Flag them clearly in the MR description and label `semver::minor` (pre-1.0) or `semver::major` (post-1.0).

## Development process

### Branching

- **Never commit directly to `main`.** Every change, even a typo fix, lands on a feature branch and gets merged via a PR/MR.
- **Branch naming**: `feature/<slug>` for new features, `fix/<slug>` for bug fixes, `chore/<slug>` for maintenance, `refactor/<slug>` for restructures, `docs/<slug>` for pure docs changes.
- **One logical change per branch.** If you find yourself wanting to do two unrelated things on the same branch, split them.
- **Rebase, don't merge.** If `main` advances while a branch is open, rebase rather than merging `main` back in.
- **Delete merged branches** after the PR/MR lands.

### Versioning (label-driven, automated on merge)

This plugin is installed from a marketplace, which means Claude Code caches it in a **versioned** directory. A change that doesn't bump the version is a change that existing installs will silently ignore when they run `/plugin update`. **Version bumps are how users actually receive your work.**

Bumps are **automated**. A GitLab CI job (`release-on-merge` in `.gitlab-ci.yml`) reads a scoped label on the merged MR, bumps `plugins/jfdi-agents/.claude-plugin/plugin.json`, commits the bump to `main` as the release bot (via `--push-option=ci.skip`), and tags `vX.Y.Z`. The tag push then triggers `publish-to-github`, which publishes a clean-room snapshot to `github.com/dr-zog/jfdi-agents` under a separate identity. **You (or Claude) do not bump the version manually.** You pick a label.

- **Where the version lives**: `plugins/jfdi-agents/.claude-plugin/plugin.json`, the `version` field. Only there. Do not set it in the marketplace entry — if both are set, `plugin.json` wins, and having two sources creates drift. **Do not edit this field on feature branches** — the release bot owns it. If your MR modifies `plugin.json` version manually, the bot's bump will collide; let the automation do its job.
- **Every MR must have exactly one semver scoped label** before it is merged. The four labels:
  - **`semver::patch`** — bug fixes, typo corrections, small clarifications to existing agent bodies or docs. Anything a user wouldn't notice at the capability level.
  - **`semver::minor`** — new agents, new skills, new commands, new shared-doc sections, changes to the TeamLead state machine or the JFDI workflow stages. Anything that adds observable capability.
  - **`semver::major`** — breaking changes: renaming an agent, removing a role, changing a skill's namespace, changing the install surface, or declaring the plugin's design stable at 1.0.
  - **`semver::none`** — MRs that should *not* cut a release: CI/infrastructure changes, README polish outside the plugin dir, marketplace-shell edits, eval tweaks. The `release-on-merge` job reads this and exits cleanly.
- **Pre-1.0 leniency**: during the `0.x` phase, `semver::minor` MRs are allowed to contain small breakages. After 1.0, `semver::major` is required for any breaking change.
- **Missing label = pipeline fails loudly.** The `release-on-merge` job aborts with a clear message if no semver label is present. Add the label from the GitLab UI and re-run the pipeline. This is deliberate — silent no-bump merges are the failure mode the automation exists to prevent.
- **If you need to correct a release** (wrong label picked, bot commit needs to be undone), the recovery is: revert the bot's bump commit, delete the bad tag (local + remote, GitLab + GitHub), fix the label, and re-run. The pipeline is idempotent against the "already released" case via its `LAST_AUTHOR == BOT_NAME` guard.

### Merge Requests

- Every branch lands via a GitLab MR, even if the reviewer is the same human who authored the change. The MR produces a durable record of *why* a change happened and is the unit the release pipeline keys off.
- MR titles should be short and present-tense imperative. The MR body is where the "why" goes.
- MR commits may be squashed on merge at the maintainer's discretion; do not rely on intra-branch commit boundaries surviving.

### Release automation — what's built

The full pipeline lives in `.gitlab-ci.yml` at the repo root and runs in three stages:

- **`mr-label-check`** (fires on MR pipelines): fails fast if no `semver::*` label is set on the MR. Catches the missing-label case at MR-open / branch-push time rather than at merge time. Hard fail by design — the fix is one click.
- **`release-on-merge`** (fires on push to `main`): locates the merged MR via the GitLab API, reads its `semver::*` label, bumps `plugin.json`, generates a new `CHANGELOG.md` entry by walking merge commits since the previous tag (via `scripts/ci-update-changelog.sh`), commits both files as the release bot with `--push-option=ci.skip`, tags `vX.Y.Z`, and pushes. Refuses to double-bump (checks the last commit author against the bot's name) and refuses to overwrite an existing tag.
- **`publish-to-github`** (fires on tag push matching `v<MAJOR>.<MINOR>.<PATCH>`): verifies the tag matches `plugin.json`'s version, **cross-builds the team-inspect binaries** for six OS/arch pairs into `plugins/jfdi-agents/bin/team-inspect/<os>-<arch>/`, `rsync`-stages the repo (excluding `PUBLISH_EXCLUDES` — the freshly-built binaries are inside the rsync-included tree), clone-replaces the public GitHub repo's tracked contents, and commits a single "Release vX.Y.Z" commit under a separate identity. Idempotent. Image is `golang:1.26-alpine` so the Go toolchain is available; binaries exist only on the GitHub mirror, never committed back to GitLab (gitignored locally).

Together the three stages mean a labelled MR merge is the only thing a human needs to do to ship a release to `github.com/dr-zog/jfdi-agents` with an updated CHANGELOG.

### CHANGELOG mechanics

`CHANGELOG.md` lives at the repo root and follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) format. Entries from `0.4.3` onward are auto-generated by `release-on-merge`:

- Section header derives from the bump label: `semver::patch` → `### Fixed`, `semver::minor` → `### Added`, `semver::major` → `### Changed`. The maintainer can manually re-categorise after the release lands if the auto-mapping is wrong.
- Each merge commit since the previous tag contributes a bullet: `- <MR title> (!N)`. MR titles are extracted from the merge commit body's first non-empty line; IIDs from the `See merge request …!N` line GitLab adds.
- `semver::none` MRs still appear as bullets in the next release's entry — they're things that happened between releases, even if they didn't cut their own release. The maintainer can edit them out before the next release if desired.
- Entries from `0.3.0` through `0.4.2` were backfilled by hand when the CHANGELOG was introduced.

If for some reason the auto-generation can't find merges in the range (rare — direct-to-main commits don't appear in `git log --merges`), the entry contains a placeholder bullet flagging the situation for manual fix.

### Release automation — what's still on the roadmap

*(Empty for now — CHANGELOG generation and MR-label check both shipped. Future enhancements get added here.)*

### Commit messages

These conventions don't drive version bumps anymore (labels do that) but they still make the commit log readable and position the repo to regenerate changelogs from history later.

- Start the title with a conventional-commit prefix: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`.
- A breaking change is marked by appending `!` to the prefix (`feat!:`) and including a `BREAKING CHANGE:` footer describing the break.
- The title is imperative and under 72 characters.
- The body is wrapped at ~72 characters and explains *why*, not *what* (the diff already shows the what).
