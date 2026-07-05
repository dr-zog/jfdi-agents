# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and this project adheres to [Semantic Versioning](https://semver.org/).

Entries from `0.4.3` onward are generated automatically by the `release-on-merge` CI job from merged MR titles since the previous tag. See `CLAUDE.md` § *"Release automation — what's built"* for the mechanism. Entries from `0.3.0` through `0.4.2` were backfilled by hand when the changelog was introduced.

## [Unreleased]

## [0.8.2] - 2026-07-05

### Fixed

- feat: align with Claude Code v2.1.178+ single-session-team model (!14)
- fix(team-lead): tighten choreography — bare idles ignored, never ask for status (!15)

## [0.8.1] - 2026-05-14

### Fixed

- fix(team-lead): trust idle teammates; slow poll loop to 120s (!13)

## [0.8.0] - 2026-05-14

### Added

- feat: Tasks for state, SendMessage for nudges; TeamLead self-loops via ScheduleWakeup (!12)

## [0.7.1] - 2026-05-08

### Fixed

- docs: clarify dr-zog/ai-marketplace tracks latest, no version pinning (!11)

## [0.7.0] - 2026-05-07

### Added

- fix(bootstrap): restore ~/.claude isolation; migrate marketplace to dr-zog/ai-marketplace (!10)

## [0.6.1] - 2026-05-06

### Fixed

- fix(ci): remove the gitignore-then-force contradiction for binaries (!9)

## [0.6.0] - 2026-05-06

### Added

- feat(team-inspect): implement list and inspect subcommands (!7)
- ci: distribute team-inspect prebuilt binaries via publish-to-github (!8)

## [0.5.0] - 2026-05-05

### Added

- ci: auto-generate CHANGELOG.md on release; mr-label-check at MR-time (!5)
- feat(bootstrap): auto-config via settings.json — `claude` Just Works (!6)

## [0.4.2] - 2026-04-30

### Fixed

- bootstrap: make initial commit when HEAD is unborn (!4)

## [0.4.1] - 2026-04-30

### Fixed

- stall-detector now respects `CLAUDE_CONFIG_DIR` (!3)

## [0.4.0] - 2026-04-29

### Added

- `write-solo-agent` skill: a third agent invocation pattern for main-session agents that pair one-to-one with a human, outside the team-of-agents flow. The skill is human-gated — Architect can propose but only the human can authorise minting. Includes the two-licensed-paths principle, the mechanism-vs-contract distinction for tests, and the integration discipline for solo work that needs to land cleanly with subsequent team sessions. (!2)
- `docs/solo-agents.md` satellite document, sibling of `team-lead-playbook.md`. Process.md trimmed to a short three-pattern overview pointing at the satellites for detail.

## [0.3.0] - 2026-04-27

Initial public release of the **JFDI pattern**.

### Added

- **JFDI workflow.** ProductOwner + Architect intake → Architect mints per-layer developer agents into `.claude/agents/` → team walks the stack sequentially for the walking skeleton → parallel refinement against a numbered acceptance list. Deliberate departure from the ATDD/Gherkin-based pattern that bogged down in up-front ceremony.
- **Six-role team.** `team-lead`, `product-owner`, `architect`, `verifier`, `repo-steward` (plugin-shipped), plus a family of Architect-minted `<layer>-dev` developer agents per project.
- **Skills.** `bootstrap`, `start`, `write-agent`, `write-team-lead`, `commit-as-agent`.
- **Per-project isolation.** `CLAUDE_CONFIG_DIR=$PWD/.claude-state` pattern. Bootstrap warns when isolation is not active. TeamLead's zombie-recovery routine refuses to `rm` cross-project state.
- **Output style.** `jfdi-agent` strips Claude Code's coding-focused defaults so role bodies start from a clean slate. Code-writing agents (the minted developers) include coding discipline explicitly.
- **GitLab CI release pipeline.** `release-on-merge` + `publish-to-github`: a labelled MR merge bumps `plugin.json`, tags `vX.Y.Z`, and publishes a clean-room snapshot to `github.com/dr-zog/jfdi-agents` under a separate identity.
- **Two evals.** `small-task-list` and `edtech-multitenant`. Tier 1 (human pastes kickoff into a TeamLead session). Scorers cover process compliance and acceptance behaviour.
- **`tools/team-inspect/`.** Go binary scaffold for inspecting Claude Code agent-team filesystem state. Subcommands stubbed; implementation pending (issues #6, #7).

### Changed

- Plugin renamed `atdd-agents` → `jfdi-agents`. Install command is now `/plugin install jfdi-agents@jfdi-agents`.
- All URLs aligned to GitHub (`homepage`, `repository`, README install command, `local-install` docs, Go module path) so the published GitHub mirror has no `gitlab.home.andrewdicks.co.uk` references. (!1)
- Publish excludes lifted into a YAML variable (`PUBLISH_EXCLUDES`) so `evals/`, `tools/`, `scripts/`, and maintainer-only docs stay out of the GitHub publish.

### Removed

- Old ATDD/Gherkin vocabulary: `cycle`, `feature` (as a planning unit), `scenario`, `step`, `roadmap`, `three-amigos` discuss meetings, `feature brief`, `mediator` role. The terminology doc explicitly bans these.
- `Planner`, `ScenarioAuthor`, `Mediator` agents — folded into the new six-role team.
