---
name: start
description: Orient the user in the jfdi-agents workflow and give them the exact command to launch the TeamLead. Reports where the project is in the Vision → Architecture → Build → Refine pipeline, verifies the agent-teams env var is set, detects whether the plugin is loaded in installed-mode or dev-mode, and prints the launch command + operating-mode prompt the human pastes into their next session. Use this skill whenever a user asks "where do I start?", "what's next?", "how do I kick off the team?", or expresses any intent to begin or resume the jfdi-agents workflow — even if they don't name the TeamLead explicitly. This skill never starts the TeamLead itself (Claude Code only lets a session pick its main-thread agent at launch); it produces the shell command the human runs after `/exit`.
---

You are guiding the user into the `jfdi-agents` workflow. Your job is to read the project state, verify launch prerequisites, and hand the user the exact command + paste-prompt for launching the TeamLead. The TeamLead is the single supported entry point — every stage of the workflow is driven from it.

## Step 1: Project state survey

One idempotent survey, one shot. Do not ad-hoc `ls` paths that might not exist — `ls` on a missing path exits non-zero and noisy errors mask real signal.

```bash
for p in vision/overview.md vision/acceptance.md docs/architecture.md docs/decisions.md vision docs .claude/agents; do
  if [ -e "$p" ]; then echo "exists: $p"; else echo "missing: $p"; fi
done
ls .claude/agents/ 2>/dev/null | grep '\-dev\.md$' || echo "no minted developers"
ls docs/demos/ 2>/dev/null || echo "no demos"
```

Read the output. The presence of `vision/`, `docs/`, and `.claude/agents/` together signals the project has been bootstrapped. If all three are missing, the project has not been bootstrapped — Step 4's output should redirect the user to `/jfdi-agents:bootstrap` rather than the TeamLead.

## Step 2: Verify agent teams are enabled (required)

The plugin requires Claude Code agent teams. The TeamLead adds specialists as named teammates and uses `SendMessage` to relay human answers back to them during multi-turn interactions.

```bash
[ "$CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS" = "1" ] && echo "enabled" || echo "disabled"
```

If `disabled`, **stop and tell the user to enable it before launching.** Bootstrap writes this to `.claude/settings.json` automatically; re-running `/jfdi-agents:bootstrap` is the cleanest fix.

## Step 3: Detect install mode (dev vs installed)

```bash
if claude plugin list 2>/dev/null | grep -q "jfdi-agents"; then
  echo "installed"
else
  echo "dev"
fi
```

If `dev`, every `claude --agent ...` line you print in Step 5 has to include `--plugin-dir <absolute-path>`, and append a one-line pointer at `${CLAUDE_PLUGIN_ROOT}/docs/local-install.md` for the install procedure that gets rid of the flag permanently.

## Step 4: Locate the current stage

| Signal | Current stage | TeamLead's first action when launched |
|---|---|---|
| `vision`, `docs`, `.claude/agents` all missing | **Not yet bootstrapped** | Asks the user to `/exit` and run `/jfdi-agents:bootstrap` first |
| Bootstrapped, `vision/overview.md` missing | **Stage 1 — Intake** | Spawns ProductOwner for the intake interview |
| `vision/overview.md` exists, `docs/architecture.md` missing | **Stage 2 — Architecture & team design** | Spawns Architect + ProductOwner |
| `docs/architecture.md` exists, `.claude/agents/*-dev.md` count is below the Developer roster section | **Stage 2 continuing — minting developers** | Architect finishes minting |
| Minted developers exist, no `docs/demos/*skeleton*` | **Stage 3 — Build (walking skeleton)** | Spawns the first layer's developer; Architect shepherds |
| `docs/demos/<date>-skeleton-complete.md` exists with Ready-to-advance: Yes | **Stage 4 — Refine** | Spawns parallel developers for the next pass |
| Acceptance list fully green on the most recent demo | **Project complete** | Asks whether to add more acceptance items or exit |

## Step 5: Print the launch block

Print the block below to the user. Tailor:

- The state lines from Step 1.
- The "Stage" and "First TeamLead action" lines from Step 4.
- The `claude` command form (installed vs dev) from Step 3.
- The operating-mode block: if the user expressed an autonomy preference (phrases like "autonomous", "full auto", "no pauses", "JFDI"), give the **Autonomous JFDI** prompt verbatim. Otherwise default to **Checkpointed**. For Autonomous JFDI, pick the **Vision-missing** variant if `vision/overview.md` does not yet exist (intake needs the human live) — otherwise use the **Vision-present** variant.

```
================================================================
jfdi-agents — ready to launch
================================================================

State:
  Vision:        <present | missing (TeamLead will run intake)>
  Architecture:  <present | missing>
  Developers:    <N minted | none yet>
  Latest demo:   <docs/demos/<file> | none yet>
  Agent teams:   <enabled (required)>
  Install mode:  <installed | dev (--plugin-dir required)>

Stage:                  <name from Step 4>
First TeamLead action when launched:
  <e.g. "Spawn ProductOwner for Vision intake (interactive via relay)"
        or "Spawn Architect + ProductOwner for architecture, acceptance list, and developer minting"
        or "Open Build phase — spawn data-dev-skeleton">

To start, /exit this session and from your terminal run:

    (installed mode, project bootstrap-ed)
    claude
    # the .claude/settings.json bootstrap wrote declares
    # jfdi-agents:team-lead as the default agent, so no flag needed.

    (installed mode, no bootstrap yet — agent flag is the override)
    claude --agent jfdi-agents:team-lead

    (dev mode)
    claude --plugin-dir "<absolute-path-to-plugin-checkout>" \
           --agent jfdi-agents:team-lead

The TeamLead's first action will be to ask your permission to create a
Claude Code agent team. Confirm with `Proceed`. The TeamLead becomes the
team lead for the rest of the session and spawns specialists as named
teammates as the workflow requires.

After the team is created, prompt the TeamLead with one of:

    Checkpointed mode (default — safer, pauses between stages):

        Proceed in Checkpointed mode. Drive <next stage>, then continue
        through the workflow, pausing for my approval after each stage.

    Autonomous JFDI mode (no pauses):

        (Vision-missing variant — use if vision/overview.md
        does not yet exist. The intake itself is an interactive
        interview; autonomy begins once Vision is captured.)

        Drive the Vision intake with me, then continue in Autonomous
        JFDI mode through architecture, the walking-skeleton build,
        and the first refine pass. Stop only on a specialist blocker
        or when the acceptance list is fully green.

        (Vision-present variant — use if Vision already exists.)

        Run in Autonomous JFDI mode. Drive <next stage> to completion
        and continue until the acceptance list is fully green. Stop
        only on a specialist blocker.

The two mode names are deliberate. Checkpointed means "I'll catch shortcuts";
Autonomous JFDI means "you can't catch them, so don't take any." Every step
(sequential-skeleton rule, Verifier sign-off, folder ownership) is
non-negotiable in Autonomous JFDI — the lack of a human checkpoint makes
the process rigour *more* important, not less.

In Checkpointed mode, the TeamLead spawns a specialist, waits for it to
finish, verifies the artefacts on disk, then asks you "Proceed?" before
moving on. You can walk away for 10–20 minutes between checkpoints.

In Autonomous JFDI mode, the TeamLead only stops on blockers, on Architect
escalations to the human, or when the acceptance list is fully green.

The TeamLead never authors artefacts itself — it only conducts.
================================================================
```

## Step 6: Brief the user on terminology if this is their first time

If no `docs/architecture.md` exists yet, this is the first or second session after bootstrap. Add a short pointer at the end of your response:

> **First time?** The team uses a strict planning vocabulary — Vision, Architecture, Layer, Acceptance list, Walking skeleton, Refine pass — documented in the plugin at `${CLAUDE_PLUGIN_ROOT}/docs/terminology.md` (the `${CLAUDE_PLUGIN_ROOT}` resolves to the plugin's install dir at runtime, not anywhere in your project). Every agent in the team speaks this language. Worth a five-minute read.

## Hard rules

- **Do not start the TeamLead yourself.** It is tempting to spawn it as a subagent of the current session via the `Agent` tool. Don't. The TeamLead is designed as a long-lived main-session agent — subagent invocations time out and return summaries, throwing away the state machine the TeamLead depends on. If the user asks "can you just run it for me right now?", politely refuse and redirect them to `/exit` and the shell command.
- **Do not suggest running any specialist directly.** `claude --agent jfdi-agents:product-owner` (or any other specialist) is not supported — `AskUserQuestion` is disallowed on every specialist, so they cannot conduct an intake without the TeamLead relay. There is exactly one entry point.
- **Do not modify any files.** This skill is read-only state reporting + a launch handoff.
- **Do not block on missing prerequisites other than agent teams.** A missing Vision is not a blocker — the TeamLead handles it. Only `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS != 1` is a hard stop.
- **Do not override the user's choice of mode.** If they want Autonomous JFDI, give them that prompt verbatim.
