# skills

A collection of Claude Code skills, distributed as a
[plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces).

Each plugin lives in its own directory containing `.claude-plugin/plugin.json`
(the manifest), `skills/<name>/SKILL.md` (the skill itself), and optionally
`DESIGN.md` (rationale and decisions behind it). The marketplace manifest at
`.claude-plugin/marketplace.json` lists every plugin.

## Installation

Inside any Claude Code session, add the marketplace once, then install the
plugins you want:

```
/plugin marketplace add dmytrovskyi/skills
/plugin install ship@dmytrovskyi
```

The skill is available immediately in every project on your machine. Verify
with `/skills` (it should be listed) or invoke it directly, e.g. `/ship`.

### Stay up to date

```
/plugin marketplace update dmytrovskyi
```

pulls the latest version of this repo and updates installed plugins — no
manual copying or symlinking.

### Local development

To test changes to a skill before pushing, run Claude Code with your working
copy loaded directly:

```bash
claude --plugin-dir ~/Projects/skills/ship
```

## Skills

| Skill | Purpose |
|---|---|
| [ship](ship/skills/ship/SKILL.md) | Autonomous delivery pipeline: plan review (2 lenses) → implementation → PR review/fix loop with per-round reports |

## General flow

The delivery flow has exactly two human gates; everything between them runs
autonomously, with a compact status report after every phase and review round:

```
 you (gate 1)          /ship — autonomous pipeline               you (gate 2)
┌─────────────┐   ┌───────────────────────────────────────┐   ┌──────────────┐
│ brainstorm  │   │ Phase 1  plan review gate (2 lenses)  │   │ read final   │
│ the design, ├──▶│ Phase 2  implementation (TDD/commits) ├──▶│ report,      │
│ write the   │   │ Phase 3  PR + review/fix loop         │   │ approve and  │
│ plan        │   │ Phase 4  handoff report, then STOP    │   │ merge the PR │
└─────────────┘   └───────────────────────────────────────┘   └──────────────┘
```

1. **Gate 1 — design (human).** Start a task as usual (Jira ticket or a task
   described in chat), answer the brainstorming questions, and let the
   planning flow produce a written implementation plan.
2. **Phase 1 — plan review gate.** Exactly two review rounds with different
   lenses: round 1 checks requirements coverage against the ticket/task text,
   round 2 checks technical feasibility against the actual codebase. A fresh
   fixer agent applies accepted fixes to the plan after each round.
3. **Phase 2 — implementation.** The reviewed plan is executed task by task
   in fresh subagents — TDD per task, one commit per task, branch and commit
   naming enforced.
4. **Phase 3 — PR + review/fix loop.** The PR is created, then the loop runs:
   fresh reviewer agent posts findings as inline comments → fresh fixer agent
   verifies each finding, fixes, tests, pushes, and replies to every
   comment — until a round confirms zero findings or the round cap (default 4)
   is reached.
5. **Gate 2 — approval (human).** Read the final report, approve, and merge.
   The pipeline never merges, approves, or force-pushes.

The main session acts as a thin **orchestrator**: it only dispatches agents,
judges convergence from their structured returns, and emits the reports.
Every heavy step (review, triage, fixing) runs in a fresh subagent with no
memory of the implementation reasoning — that independence is the point.

## Skills the pipeline builds on

`ship` is an orchestration layer, not a monolith — each stage delegates to an
existing skill:

| Skill | Where in the flow | Role |
|---|---|---|
| [superpowers:brainstorming](https://github.com/obra/superpowers) | Gate 1, before `/ship` | Design Q&A that validates the spec with the human |
| [superpowers:writing-plans](https://github.com/obra/superpowers) | Gate 1, before `/ship` | Produces the implementation plan doc that `/ship` discovers and reviews |
| [superpowers:subagent-driven-development](https://github.com/obra/superpowers) (or executing-plans) | Phase 2 | Executes the plan task-by-task in fresh subagents |
| [superpowers:test-driven-development](https://github.com/obra/superpowers) | Phase 2 | TDD discipline for every implementation task |
| code-review (built-in) | Phase 3, each review round | Reviewer agent runs `code-review <effort> <pr-url> --comment` to post inline PR findings |
| [superpowers:receiving-code-review](https://github.com/obra/superpowers) | Phase 3, each fix round | Fixer agent verifies every finding in the code before acting — a reviewer claim is not a fact |
| claude-md-management:revise-claude-md | After merge | Final report reminds you to capture session learnings in CLAUDE.md |

Other dependencies: the **Atlassian MCP server** (optional — fetches Jira
tickets as the requirements source; without it, free-form task text or the
plan doc serve the same role) and the **GitHub CLI** (`gh`) for PR creation,
comment watermarking, and replies.

## Using ship

`/ship` automates the delivery flow between two human gates: you answer the
design questions before it starts, and you approve/merge the PR at the end.
Everything in between — plan review, implementation, PR creation, review/fix
rounds — runs autonomously, with a compact status report after every phase and
review round (see [General flow](#general-flow) above). It never merges,
approves, or force-pushes.

Invocations:

```
/ship                                   # full pipeline from an existing plan
/ship MA-1234                           # ticket by bare key
/ship https://.../browse/MA-1234        # ticket by URL
/ship <pr-url>                          # skip to the review/fix loop on an existing PR
/ship high                              # override review effort (low|medium|high|xhigh|max)
/ship max-rounds=2                      # cap the review/fix rounds (default 4)
/ship effort=high make retries configurable   # effort + free-text task combined
/ship add CSV export to measurements    # no ticket — the text IS the requirements
```

Requirements can come from a Jira ticket, free-form task text, or both; with
neither, the plan document itself (or, on an existing PR, the PR
title/body/diff) is used. See [ship/skills/ship/SKILL.md](ship/skills/ship/SKILL.md)
for the full procedure and [ship/DESIGN.md](ship/DESIGN.md) for the design
decisions.
