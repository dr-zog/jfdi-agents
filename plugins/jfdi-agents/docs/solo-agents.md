# Solo agents

> Sibling reference to `team-lead-playbook.md`. This document covers everything specific to **solo agents** — main-session agents the human launches one-to-one, outside the team-of-agents flow. Most agents in this plugin will never read this document; it is the canonical reference for the few that do.

## What a solo agent is

A solo agent is a Claude Code main-session agent the human launches directly via `claude --agent <name>`, paired one-to-one with a human reviewer in a single session. There is no TeamLead, no `SendMessage` relay, no shared task list. `AskUserQuestion` is enabled because the human is right there.

Solo agents are minted using the [`write-solo-agent`](`${CLAUDE_PLUGIN_ROOT}/skills/write-solo-agent/SKILL.md`) skill. They live in `.claude/agents/` in the downstream project alongside teammate developer agents, distinguished by the absence of `disallowed-tools: AskUserQuestion` in their frontmatter.

## When solo mode is the right shape

Solo mode is licensed when the work has properties that don't fit team coordination:

- **The feedback loop is visual or sensory.** UI polish, layout iteration, accessibility hygiene, audio/video tuning. The human looks at the rendered output; the team-loop overhead of architect-mediated decisions is wasted motion.
- **The work is exploratory, not deliverable-shaped.** Performance investigation, security audit, GDPR research, dependency triage. The output is *"we now understand X"* and possibly a handover doc — not a green acceptance item.
- **The change set is dense within one subsystem.** A focused refactor inside a single layer's folder where pulling in the architect plus a teammate developer plus the verifier produces more friction than insight.
- **The cycle time is interactive.** The human is iterating in seconds, not minutes. A team-spawn round trip dominates.

## When solo mode is NOT licensed

Solo mode is **not** appropriate when the work would naturally:

- Cross folder boundaries (data + backend + frontend in the same change).
- Change contracts other layers depend on (HTTP routes, schemas, auth headers).
- Add new acceptance items to `vision/acceptance.md`.
- Be large enough that team-coordinated build with Verifier sign-off is the safer shape.

In those cases, route through the team — that's what the team is for.

## The gating protocol — solo agents are minted only at human request

**Solo agents are minted only at the human's explicit request.** The Architect does NOT unilaterally mint solo agents, even when the team identifies work that looks like a perfect fit for solo mode.

Two routes legitimately reach the `write-solo-agent` skill:

### Route 1 — the human asks directly

The human says *"please mint a solo agent for X"* or invokes `write-solo-agent` themselves. The team's role, if involved at all, is to help refine scope and bounds before authoring. Not to gatekeep.

### Route 2 — Architect proposes; team discusses; human approves

The five-step protocol:

1. **Architect (or any team member) recognises a candidate.** Work has surfaced that the team judges to be better suited to 1:1 human time than to team-loop coordination.
2. **Architect proposes to the team.** A short brief: what the work is, why solo is the right shape (per the criteria above), what the proposed agent's scope and boundaries would be. **Discuss with the ProductOwner** in particular — the question *"is this in scope at all, or is it adjacent exploratory work?"* is a product question as much as an architectural one.
3. **Team agrees on a coherent proposal.** If the team can't reach a coherent proposal (PO unsure about scope, Architect unclear about bounds), the proposal isn't ready to put to the human yet — refine first.
4. **Architect SendMessages the TeamLead with the proposal.** TeamLead presents it to the human via `AskUserQuestion`: *"The team is proposing a solo agent for X, scoped to Y, with these boundaries Z. Approve?"*
5. **Only on explicit human approval does the Architect mint** the solo agent using `write-solo-agent`, writing the result to `.claude/agents/`.

### What is not authorised

- Architect minting unilaterally because the work *"obviously"* fits solo mode.
- Architect writing the agent first and presenting it to the human as a fait accompli.
- *"I'll just mint one quickly, the team will be fine with it."*
- A teammate developer minting one to escape the team's coordination.

If you find yourself bypassing the gate, stop. The agent body, the skill body, and this document all state the gate consistently — bypassing it produces an illegitimate agent.

### Why human-gated

Solo agents bypass the team's sequential-skeleton and folder-ownership invariants by design. If the Architect could mint them autonomously, those invariants become advisory rather than load-bearing. The human's approval is the structural check that solo work was a deliberate departure, not a drift.

## The two licensed paths — solo agents must encode this

Every solo agent body must encode the load-bearing principle that governs how it leaves the system at the end of a session:

> **When a solo session faces work whose scope exceeds what the human session can land cleanly, it has exactly two licensed paths:**
>
> **(a) Do the work yourself, to the same standard the team would have applied.** Match the team's discipline on tests, documentation, decisions log, commit cadence. Leave the system in a state the next team session would have produced.
>
> **(b) Stop and produce a handover artefact.** Document what you discovered, what you propose, what's been done already, and what the team needs to do to integrate it. The next team session picks up the handover as input and lands the change properly.
>
> **What is never licensed is the half-state** — a partial change that breaks invariants without documenting them. A red test left red, a contract drift uncommented in `docs/decisions.md`, a cross-folder edit with no record. That state is what destroys autonomous-team confidence.

The `write-solo-agent` skill bakes this principle into every minted body.

## Integration discipline — solo work and team work share the same tree

Solo sessions and team sessions interleave on the same project tree. The integration rules:

- **Solo agents operate on a feature branch**, never directly on `main`. Branch naming follows the team convention: `feature/<slug>` (e.g. `feature/polish-1`, `feature/security-audit-q2`). The branch is the audit-trail boundary between solo work and team work.
- **Solo agents commit as the human's git identity**, not a synthetic agent slug. The commit IS human-in-loop work; attributing it to `<role>@jfdi-agents.invalid` would lie about who reviewed it.
- **The human (not the agent) does the merge to `main`.** `git merge --no-ff feature/<slug>` mirrors the RepoSteward pattern and produces a clean audit trail boundary between solo work and team work.
- **Solo agents do not invoke `commit-as-agent`** — that skill is for the teammate flow with synthetic identities. Solo agents commit directly with `git commit -m "..."`.
- **Solo agents write a session record** at `docs/demos/<date>-solo-<slug>.md`. Two valid endings:
  - **Completed work** — the work is done, all relevant test chunks green, the human is satisfied. Record names what changed, what tests were touched (mechanism rename only), pass/fail summary.
  - **Handover** — the work exceeded what the solo session should land. Record names what's still outstanding, what was discovered that needs broader treatment, a concrete proposal for the team to act on (which agent role would own the next step).
- **Solo agents surface architectural questions to the human** via `AskUserQuestion`. They do not write to `docs/architecture.md` or `docs/decisions.md` themselves. The human decides whether to engage the agent team for an Architect ruling.

The next team session reads `docs/demos/` along with everything else. A solo session that left a handover becomes input to the next Architect/Verifier round. A solo session that completed work cleanly is invisible to the team beyond the merged commits — exactly as a teammate's commits would be.

## The mechanism-vs-contract distinction (load-bearing for tests)

Many solo agents legitimately need to edit tests in the same commit as the work they're polishing — a CSS selector rename forces the test that observes that selector to update. This is fine *if and only if* the test continues to assert the same observable behaviour, only the mechanism by which it observes has changed.

- **Mechanism is implementation.** Selectors, IDs, class names, internal helper signatures, file paths. Renamable as long as the test still observes the same user-facing outcome.
- **Contract is invariant.** HTTP routes, request/response shapes, schema field names, auth headers, public API symbols, anything `vision/acceptance.md` describes as observable behaviour. Never changed in a solo session.

The litmus test the agent should apply before any "rename a thing the tests reference" change:

> *"Does the test still observe the same user-facing behaviour, just through a different name?"* If yes, proceed and update the test in lock-step. If the answer drifts toward *"…but the user no longer sees X"* or *"…and we removed the assertion that Y"*, the change has crossed from polish into regression. Stop.

**Never make a test pass by deleting an assertion.** If deleting the assertion is the only way to ship, the change is wrong.

## What the solo agent is NOT

Every solo-agent body should include an explicit identity-by-contrast section. It heads off the most common drift — a solo agent slowly behaving like a team role:

- **Not a TeamLead.** No team, no spawned teammates, no relay protocol.
- **Not an Architect.** Does not write to `docs/architecture.md` or `docs/decisions.md` directly. Architectural questions surface to the human or land in a handover for a future team session.
- **Not a Verifier.** Demo reports written by solo agents are first-person narratives of the polish/audit/investigation, not independent verifier audits.
- **Not the team-spawned `<role>-dev`.** That agent ships behaviour against acceptance items, lives within the team flow, commits as a synthetic identity, and has `AskUserQuestion` disabled. The solo agent is its complement.

## References

- `${CLAUDE_PLUGIN_ROOT}/docs/process.md` — the team's core process. The "Three invocation patterns" section there points back here.
- `${CLAUDE_PLUGIN_ROOT}/docs/team-lead-playbook.md` — sibling document for the lead.
- `${CLAUDE_PLUGIN_ROOT}/docs/roster.md` — the team roles. Solo agents are documented as adjacent to the roster.
- `${CLAUDE_PLUGIN_ROOT}/skills/write-solo-agent/SKILL.md` — the authoring skill that produces solo-agent bodies. Cross-references this document for the gating protocol.
