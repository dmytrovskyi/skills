# ship — design (approved 2026-08-13; formerly pr-cycle)

Personal automation of review/implementation into a single autonomous
pipeline with two human gates: kickoff Q&A (before) and PR approval/merge (after).
Fully autonomous between the gates; a compact report is emitted to the user after
every phase and every review round.

## Decisions made with the user

- **Reviewer isolation**: fresh subagents replace the second "Reviewer tab" — a
  subagent has no memory of the implementation reasoning, so review independence
  is preserved without a second session.
- **Plan review gate**: 2 rounds with *different lenses* (not the same review
  twice): Round 1 = requirements coverage vs the Jira ticket; Round 2 =
  technical soundness vs the actual codebase.
- **Review target**: the implementation plan (writing-plans output), not the
  design spec — the spec is already validated by the user during brainstorming.
- **PR review effort**: `medium` by default (user finds `high` token-expensive
  and nitpicky); overridable per invocation.
- **Convergence over fixed count**: PR review/fix rounds stop when a round
  yields zero confirmed findings, capped at 4 rounds (user previously did 3–5
  manually).
- **Autonomy**: no pauses between rounds; per-round reports instead.
- **What stays manual**: brainstorm Q&A, final PR approval/merge,
  /revise-claude-md (skill reminds about it in the final report).
- **Requirements source** (added 2026-08-13): a Jira ticket, inline task text
  typed in Claude Code, or both — no-ticket mode is first-class, not an
  error. With no key, branch/commit/PR templates drop the `<KEY>` segment.
- **Thin orchestrator** (added 2026-08-19, spec:
  `docs/superpowers/specs/2026-08-19-ship-thin-orchestrator-design.md`):
  supersedes the original "Worker does all triage/edits" split below. Per
  review round a fresh fixer subagent verifies findings, edits, tests,
  commits, pushes, and replies to PR comments; a plan-fixer does the same
  for the plan gate. The main session only dispatches agents, judges
  convergence/STOP from their structured returns, and reports. Motivation:
  long runs bloated the main context, and same-context triage inherits
  implementation bias.

## Architecture

One skill (`ship`) executed by the main session as a thin **Orchestrator**:
setup/detection, one-time requirements fetch (Jira MCP lives only in main;
agents get the text pasted into their prompts), agent dispatch, convergence
and STOP decisions, and every user-facing report. After Setup it never
edits files and never reads full diffs, plan contents, or comment bodies —
agent returns map 1:1 to the report slots.

- **Reviewer subagents** (Agent tool, fresh context each round):
  - Plan Round 1: plan doc + requirements → requirements findings
  - Plan Round 2: updated plan + repo → feasibility findings
  - PR rounds: run `code-review <effort> <pr-url> --comment` (posts inline
    PR comments itself)
- **Fixer subagents** (fresh context each round):
  - Plan rounds: verify findings vs plan + requirements, edit the plan doc,
    round 2 makes the doc commit
  - PR rounds: fetch past-watermark comments, verify each finding
    (receiving-code-review rigor), fix, test, commit, push only on green,
    reply to every round comment; guardrails travel verbatim in the prompt

Entry modes: plan exists → Phases 1–4; PR already exists → Phases 3–4.

## Report contract

After each phase/round: phase name, findings raised → confirmed → fixed /
rejected (with one-line reasons), commits pushed, next action. ≤ 12 lines.

## Out of scope

- Merging the PR (never).
- Cloud/cron unattended runs (Atlassian MCP auth may be absent headless).
- Workflow-tool fan-out review (possible later upgrade if medium reviews
  produce too many false positives).
