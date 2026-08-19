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

## Using ship

`/ship` automates the delivery flow between two human gates: you answer the
design questions before it starts, and you approve/merge the PR at the end.
Everything in between — plan review, implementation, PR creation, review/fix
rounds — runs autonomously, with a compact status report after every phase and
review round. It never merges, approves, or force-pushes.

Typical flow:

1. Start a task as usual (Jira ticket or a task described in chat), go through
   brainstorming, and let the planning flow write the implementation plan.
2. Run `/ship`.
3. It reviews the plan twice (requirements lens vs the ticket, then technical
   lens vs the codebase), implements the plan (TDD, commit per task, branch
   and commit naming enforced), opens the PR, then loops: fresh reviewer
   agent → verify findings → fix → push → reply to every comment — until a
   round confirms zero findings or the round cap is reached.
4. Read the final report, approve, and merge.

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
