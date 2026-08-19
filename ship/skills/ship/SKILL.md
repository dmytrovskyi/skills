---
name: ship
description: Use when an implementation plan is ready and the work should proceed to code and a reviewed GitHub PR, or when an existing feature branch/PR needs automated review-and-fix rounds before the user approves the merge.
---

# ship — plan review → implement → PR review/fix loop

## Overview

Autonomous delivery pipeline between two human gates. The user already answered
design questions (gate 1); the user approves/merges the PR at the end (gate 2).
Everything between runs without pausing, but a **status report is emitted to the
user after every phase and every review round** — the bold "Emit a report"
markers below are the complete list; phases don't get an extra summary on top
of their round reports.

Roles: the main session is the **Orchestrator** — after Setup it only
dispatches agents, collects their structured returns, judges convergence and
STOP conditions, and emits every report; it never edits files and never
reads full diffs, plan contents, or comment bodies. Every heavy step runs in
a fresh subagent: reviews in **reviewer agents** (no memory of the
implementation reasoning — that independence is the point; never review your
own work inline), triage-and-fix in **fixer agents** (fresh relative to both
the implementation and the reviewer — the next round's reviewer is the check
on a fixer's work). Agents share no conversation or MCP context: everything
an agent needs is pasted into its prompt, and everything the Orchestrator
needs back comes in the agent's return, shaped to fill the report slots 1:1.

## Arguments

`/ship [effort] [max-rounds=N] [pr-url] [jira-url] [task text]` — all
optional.

| Arg | Default | Meaning |
|---|---|---|
| effort | `medium` | Effort for code-review: one of `low`, `medium`, `high`, `xhigh`, `max` (also accepted as `effort=<level>`) |
| max-rounds | `4` | Cap on PR review/fix rounds; minimum 1 — clamp `0`/negative to 1 and note it in the first report |
| pr-url | auto-detect | Target PR for Phase 3 |
| jira-url | auto-detect | Ticket URL |
| task text | conversation | Free-form task description; drives requirements, plan discovery, branch title |

A token binds positionally only when it matches its form — `max-rounds=N`,
URLs by shape, a bare ticket key (e.g. `MA-1234`) as the ticket. A leading
effort word binds ONLY when every remaining token also parses as a
recognized arg; if free text follows, the whole string is task text (so
`/ship max out the retry limit` is all task text, not effort=`max`). To
combine effort with task text, use the keyword form:
`/ship effort=high make retries configurable`. A task described in the conversation before invoking `/ship`
counts the same way.

## Setup (runs for every entry mode)

- **Clean tree first**: check `git status --porcelain` before any checkout,
  branch creation, or rename. If the tree is dirty, stop and ask (stash,
  commit, or include?) — silently carrying uncommitted edits onto the work
  branch would sweep unrelated changes into a later fix commit.
- **Requirements source** — a Jira ticket, inline task text, or both:
  - Jira key: run entry-mode/PR detection FIRST (same ordering rule as the
    plan-doc bullet). An explicit `jira-url` argument (or bare key) wins;
    else parse (e.g. `MA-1234`) from the branch name — which in existing-PR
    mode means the PR's HEAD branch (`gh pr view <pr> --json headRefName`),
    never whatever branch the session happened to be on; else from the plan
    doc AFTER it is discovered (discovery below does not require the key —
    no circularity). When sources disagree, follow that order and note the
    mismatch in the first report.
  - Key found → fetch via Atlassian MCP `getJiraIssue` (get the site via
    `getAccessibleAtlassianResources`; browse URL =
    `https://<site>/browse/<KEY>`). If the fetch fails for any reason, report
    that and continue with the task text or the plan doc; if neither exists,
    stop and ask.
  - No key anywhere → the user's task text IS the requirements; that is a
    normal mode, not an error. In existing-PR mode the PR title, body, and
    diff are themselves a requirements source — with no ticket and no task
    text, review against those; never stop and ask for requirements in that
    mode. Stop and ask only in full-pipeline mode when there is no ticket,
    no task text, and no plan doc to work from.
  - Both exist → the ticket is the primary requirements; the task text adds
    constraints on top.
  - Neither exists but a plan doc does (bare `/ship` on a repo with one
    recent plan) → the plan doc's goal/requirements section IS the
    requirements source for the Phase 1 requirements lens, and the branch
    title derives from the plan title.
- **Plan doc**: run PR detection (the entry-mode check below) FIRST. If an
  open PR was AUTO-detected, still run a cheap existence probe (list the
  plans directory, nothing more) — its result feeds the stale-PR sanity
  check in the entry-mode block, which would otherwise never have inputs.
  Beyond that probe, existing-PR mode skips discovery — it never reads a
  plan doc and must never pause to ask about one. Only with no open PR,
  discover the plan from candidates under the repo's plans directory
  (`docs/plans/`, `docs/superpowers/plans/`, or wherever writing-plans put
  it), newest first. Narrow by whatever is already known — ticket key in
  name/content if a key exists, topic match against task text if there is
  any — but neither is required: a single newest candidate wins on its own,
  and the key may then be extracted FROM the chosen doc. Ask only when more
  than one candidate remains.
**Entry mode** (decide this BEFORE touching any branch):
- An OPEN PR exists — explicit `pr-url`: verify with `gh pr view <pr> --json
  state,headRefName` (non-OPEN → stop and ask); auto-detect: probe with
  `gh pr list --head <current-branch> --state open --json number` — an empty
  result or probe error just means "no PR", it is NOT a gh error under
  Failure handling; closed/merged PRs never count. Sanity-check auto-detected
  (not explicit) PRs before committing to this mode, using signals gathered
  BEFORE the mode is fixed: the plans-directory existence probe (see the
  Plan doc bullet), any task text, and a ticket key parsed from the
  SESSION's current branch (the PR-head rule applies only after this mode is
  confirmed). If any of those does NOT correspond to the detected PR's
  branch or title — e.g. a fresh plan doc newer than yesterday's still-open
  PR on the branch you stayed on — fall through to "ask the user which mode"
  instead → run Phases 3 → 4 only. Check out the PR's head branch
  (`gh pr checkout <n>`); never create or rename branches in this mode — a
  non-conforming name is only flagged in the next report.
- Else a plan doc was found and the session is on the default branch, or on
  a feature branch with no commits ahead of the default branch other than
  docs/plan commits → run Phases 1 → 2 → 3 → 4, applying the branch
  convention below first. (The default branch itself always qualifies — the
  "no implementation commits" test is measured against the base, never
  against the branch's full history.)
- Anything else (no plan and no PR, two candidate plan docs, code committed
  but no PR) → ask the user which mode to run; don't guess.

**Branch convention** (full-pipeline mode only): the working branch must match
`(fix|chore|feat)/<KEY>-<task-title>`, where `<task-title>` is the ticket
title with spaces replaced by hyphens, original casing kept (e.g.
`fix/MA-4367-Good-to-know-in-Mapping-Summary-does-not-show-MVA-values`);
type chosen from the nature of the work (bug → fix, feature → feat,
maintenance → chore). With no ticket key, drop the `<KEY>-` segment:
`(fix|chore|feat)/<task-title>`, title derived the same way from a short
task summary. Sanitize the title into a valid git ref: replace any character
outside `A-Za-z0-9._-` with a hyphen, collapse repeated hyphens AND repeated
dots (git refuses `..`), cap at ~60 characters, then AFTER the cap strip
leading/trailing hyphens and dots and any trailing `.lock` (git refuses
both a trailing `.` and a component ending in `.lock`).
If currently on the default branch, create the branch from this template. If on a non-conforming branch that was never pushed, rename
it (`git branch -m`) to conform. If it is already pushed, do NOT rename —
flag the mismatch in the next report and continue.

## Phase 1 — Plan review gate (exactly 2 rounds, different lenses)

**Round 1 — requirements lens.** Spawn a fresh reviewer subagent. Its prompt must
contain: the plan doc path; the full requirements pasted in — ticket
summary/description/acceptance criteria and/or the user's task text
(subagents don't share your MCP or conversation context); the job — verify
every requirement maps to a plan step, list missed edge cases, flag scope
creep; the output contract — findings list (title, severity, evidence); and
"do NOT edit any files".

Then spawn a fresh **plan-fixer** subagent — the Orchestrator never triages
or edits the plan itself. Its prompt must contain: the plan doc path; the
same full requirements pasted in; the reviewer's findings verbatim; the
job — verify each finding against the plan and requirements before acting
(a reviewer claim is not a fact), apply accepted fixes to the plan doc,
reject the rest; "do NOT commit"; and the return contract — per finding,
accepted-and-applied or rejected with a one-line reason, plus which plan
sections changed. **Emit a report** filled from that return.

**Round 2 — technical lens.** Spawn a fresh reviewer subagent with the *updated* plan
doc path and the repo root path, instructed to read the codebase. Its job:
check every referenced file/function/pattern actually exists as described;
steps are feasible and correctly ordered; each task specifies its test. Same
output contract, no edits.

Then spawn a fresh plan-fixer subagent with the same prompt shape (updated
plan doc path, requirements, round-2 findings, same return contract) plus
two additions: the ticket key (or "none"), and the job of committing the
plan updates from BOTH rounds — `docs: apply plan review fixes (<KEY>)`;
omit `(<KEY>)` with no ticket — doc commits are exempt from the
first-implementation-commit template below. Skip the commit entirely if
both rounds left the plan doc unchanged — an empty commit attempt is not a
git error, just nothing to do. **Emit a report.** Proceed — never run a
third plan round; anything still wrong surfaces in Phase 3 against real
code, where it is cheaper to judge.

## Phase 2 — Implementation

Execute the reviewed plan with the superpowers flow already in use
(**REQUIRED SUB-SKILL:** superpowers:subagent-driven-development, or
superpowers:executing-plans if the plan says so). TDD per task, commit per
task. The FIRST IMPLEMENTATION commit on the branch (docs/plan commits don't
count) must be `<type>: <KEY> <task title>` — `<type>` is the branch prefix
when it is one of `fix|chore|feat`, otherwise derived from the nature of the
work (same rule as branch creation); no ticket → `<type>: <task title>`.
Implementer subagents don't know this rule, and under parallel dispatch
"whichever commits first" is unknowable at spawn time — so SERIALIZE the
first plan task: dispatch it alone with the required message verbatim in its
prompt and wait for its commit to land. Remaining tasks are dispatched
sequentially by default: parallel implementers sharing one working tree
corrupt each other (index.lock contention, `git add -A` sweeping another
agent's half-written files, tests reading files mid-edit). Parallelize only
with per-agent worktree isolation (Agent tool `isolation: "worktree"`) AND
tasks the plan marks as touching disjoint files. When the
plan is fully executed and tests pass, **emit a report** (tasks completed,
test command + result, commits). As in the Phase 3 loop, "tests pass" means
the KNOWN test command is green; if no test command can be determined (no
suite, docs/config-only plan), that is "tests not run", NOT red tests —
proceed and say so in the report.

## Phase 3 — PR + review/fix loop

**Create the PR** if none exists: push the branch, then `gh pr create` against
the repo's default branch — title `<type>: <KEY> <summary>` (same template
and `<type>` rule as the first commit; no ticket → no KEY), body
with the ticket browse URL (or the task statement when there is no ticket),
a short change summary, and
the standard generated-with footer. If a PR exists, reuse it (verify the
branch is pushed and up to date). **Emit a report** with the PR URL. Comments
already on the PR at this moment (earlier humans or bots) are NOT triaged by
the loop — list them in this report for the user instead.

**Loop (round = 1 … max-rounds):**

1. **Watermark.** Record TWO watermarks — the highest review-comment `id`
   and the highest issue-comment `id` (0 if none) — WITHOUT loading comment
   bodies into the Orchestrator's context:
   `gh api --paginate repos/{owner}/{repo}/pulls/{n}/comments --jq '.[].id' | sort -n | tail -1`
   and the same for `issues/{n}/comments` (empty output = 0). The two
   endpoints have independent id sequences; never compare across them. Ids
   are assigned by GitHub and monotonic — immune to local clock skew,
   unlike timestamps.
2. **Review.** Spawn a fresh reviewer subagent whose task is to invoke the top-level
   `code-review` skill via the Skill tool, args `<effort> <pr-url> --comment`
   (`--comment` makes it post findings as inline PR comments itself). Put the
   requirements context in the subagent's prompt — the ticket browse URL if
   one exists, plus a one-line task summary — not in the skill args. Require
   the subagent to end its reply with a findings summary: a count plus one
   line per finding (`path:line — title`). That returned summary is the
   authority for the round: `review_ran` is true iff the subagent completed
   and returned it — a summary with count 0 is a valid, clean round. PR
   comments prove nothing either way (the reviewer may fall back to printing
   findings without posting, and other people post comments too). If the
   summary reports zero findings, skip step 3 — there is nothing to fix.
   Before reporting, re-run the step-1 id one-liners: if any id now exceeds
   its endpoint's watermark, fetch just those comments and list them in the
   report as unrelated comments — the one sanctioned exception to the
   Orchestrator's no-comment-bodies rule (they are not findings and do not
   affect convergence).
3. **Fix.** Spawn a fresh **fixer** subagent — the Orchestrator never
   triages, edits, pushes, or replies itself. Its prompt must contain:
   - the PR URL, the repo root path, and both step-1 watermarks;
   - the reviewer's findings summary verbatim;
   - the requirements context — the ticket browse URL if one exists, plus
     a one-line task summary;
   - the Phase 2 test command; in PR-only mode instead: "derive the test
     command from the repo (package.json scripts, Makefile, CI config)";
   - these guardrails verbatim: "Never merge, approve, or close the PR.
     Never force-push. If the KNOWN test command fails after your fixes,
     commit the applied fixes locally, do NOT push, do NOT reply to any
     comment, and return the failing output." (never strand fixes as
     uncommitted hunks that trip the Setup clean-tree gate on the next
     run);
   - the job: fetch comments with `id` greater than each endpoint's OWN
     watermark (`gh api --paginate repos/{owner}/{repo}/pulls/{n}/comments`
     plus `gh api --paginate repos/{owner}/{repo}/issues/{n}/comments`,
     filter all pages — never truncate) and match them to the findings by
     path/line/title — matches are the reply targets. A finding with NO
     matching comment is still a finding (triage it from the summary; reply
     where its comment would have been, as one issue-level comment). A
     past-watermark comment matching NO finding is not a round finding
     regardless of author (the user, the fixer, and the reviewer share one
     account, so authorship cannot distinguish them) — put it in the
     return's unrelated-comments list, untriaged. Verify each finding in
     the code first — **REQUIRED SUB-SKILL:**
     superpowers:receiving-code-review; a reviewer's proposed patch is a
     finding to verify, not code to apply verbatim. A finding that implies
     a requirement change or new scope is not implemented — reject it as
     "deferred — question for the user". Apply accepted fixes; run the test
     command and require it green BEFORE pushing; no derivable test command
     is NOT red tests — push anyway and return "tests not run — no test
     command found". Then commit (any number of commits); push; reply to
     every round comment — inline comments via the thread reply endpoint,
     issue-level comments via one summary comment — stating what changed or
     the factual rejection reason (no arguing);
   - the return contract, mapping 1:1 to the report slots: counts
     (raised → confirmed → fixed, rejected), each rejected finding with its
     one-line reason (deferred scope-change findings included), commit SHAs
     (marked local-only when not pushed), unrelated/unmatched comments,
     tests status (green, or not run
     + why, or RED + failing output), and whether the push and the replies
     happened.
4. **Emit a report** (see contract) filled from the reviewer's and (when
   step 3 ran) the fixer's returns. A RED tests status in the fixer's return is a report-and-STOP
   condition, same as red tests in Phase 2: state which commits are
   local-only and that this round's comments were NOT replied to.
5. **Converge.** Only a round marked `review_ran` (a returned findings
   summary exists) can converge: if it produced **zero confirmed findings**,
   exit the loop early — raised-but-all-rejected counts as converged (the
   rejections carry into the final report). A round with no summary must NOT
   be read as clean — continue to the next round. A round whose fixer was
   lost (see Failure handling) can never converge either, regardless of the
   reviewer's summary. Otherwise continue to the next round.

## Phase 4 — Handoff (stop here)

**Emit a final report** in the same contract shape (exempt from the 12-line
cap — its `Rejected:` list may need the room): `Changes:` = PR URL and
rounds run; `Rejected:` = all still-open or rejected findings across rounds;
`Next: waiting for your approval` plus a reminder that `/revise-claude-md` is
worth running after a productive session. If the latest pushed code was
never reviewed — the loop hit max-rounds with fixes applied in the final
round, the final round was lost, or NO round at all was marked `review_ran`
— state that explicitly; likewise if the final round's fixer was lost,
state that its findings remain unaddressed. Then STOP. Approving and merging the
PR is the user's decision — never merge, never mark the PR ready/approved.

## Report contract (every "Emit a report" above)

≤ 12 lines excluding the `Rejected:` list, which may always run one line per
rejection (Phase 4's final report is exempt entirely, see above). Exactly
these slots, headed `### Phase <n> [Round <m>] — <name>` (so Phase 1's two
reports are distinguishable: `### Phase 1 Round 1 — requirements lens`) or
`### Round <n> — PR review`:

```
Findings: <raised> raised → <confirmed> confirmed → <fixed> fixed, <rejected> rejected
Rejected: <finding — one-line reason>  (one line each; omit slot if none)
Changes: <files touched / plan sections updated / commit SHAs pushed / PR URL>
Unrelated comments: <pre-existing or unmatched PR comments surfaced for the user>  (omit slot if none)
Notes: <anything a rule says to "note/flag in the report": clamped max-rounds, source mismatches, non-conforming branch, scope questions>  (one line each; omit slot if none)
Next: <what runs next, or "waiting for your approval">
```

Phases without findings (2, PR creation) fill `Findings:` with `n/a` and use
`Changes:` for their substance. Every agent's return contract maps 1:1 to
these slots — fill reports from the returns; never re-read the diff, the
plan, or the comments to write a report. Reports go in the turn's
user-visible text — never only in thinking or tool output.

## Failure handling

- Reviewer subagent dies or returns garbage → retry once with the same prompt;
  if it fails again, skip to the **next round** (or Phase 4 if rounds are
  exhausted), noting the lost round in the report. A lost round is not
  `review_ran` and can never satisfy convergence; if the LAST round was lost,
  the Phase 4 final report must carry an explicit warning that the latest
  code was never reviewed.
- A fixer subagent (plan-fixer or PR fixer) dies or returns garbage → retry
  once with the same prompt; if it fails again the round's fixes are lost:
  the Orchestrator restores the working tree to the last commit before
  proceeding — a dead fixer's half-applied edits must never be swept into
  a later commit or trip the Setup clean-tree gate. For a PR round the
  reviewer's comments were posted but unaddressed — say so explicitly in
  the round report, continue to the next round (or Phase 4, with the
  unaddressed-findings warning), and never count the round toward
  convergence. For a lost Phase 1 plan-fixer, the restore also discards
  any earlier uncommitted plan edits — proceed with the plan as last
  committed, note the lost edits in the report, and let Phase 3 re-judge
  against real code.
- `git push` / `gh` errors, merge conflicts with base → report and STOP
  (needs the user).
- Phase 2 ends with tests not green, or a plan task that cannot be
  completed → emit a report with the failing output and STOP; never
  proceed to Phase 3 on red tests.
- A finding implies a requirement change or new scope → the fixer rejects
  it as "deferred — question for the user" (its prompt says so) and the
  Orchestrator notes it in the report; continue with in-scope work. For
  convergence it counts with the rejections (it is deferred, not
  blocking) — otherwise the reviewer re-raises it every round and the loop
  can never converge; carry it into the final report.

## Guardrails

- Never merge, approve, or close the PR. Never force-push.
- Reviewer subagents never edit anything; during review rounds only that
  round's fixer subagent edits — the Orchestrator never edits files in any
  phase after Setup. (Phase 2 implementer subagents edit code as their
  sub-skill directs — that is not a violation.)
- Fixer prompts must carry the merge/approve/force-push guardrails
  verbatim — the fixer holds push and reply powers.
- If a reviewer proposes a patch, the fixer treats it as a finding to
  verify, not as code to apply verbatim.
- Agents never converse with the user; every report comes from the
  Orchestrator.
- Never skip a per-round report because "nothing interesting happened" —
  a zero-findings round is exactly what the user needs to see to approve.
