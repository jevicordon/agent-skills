---
name: implementation-coordinator
description: Implement an ordered set of GitHub issues on the current work branch, with one agent and commit per issue pushed to a draft pull request as it goes, then verify the series and finish the pull request. Use when the user supplies an explicit list (possibly split into series), a parent issue, a milestone, or a label and wants coordinated implementation.
argument-hint: '<#12 #13 | A: #12 #13; B: #20 | parent #40 | milestone "v2" | label "export">'
disable-model-invocation: true
---

# Implementation Coordinator

Coordinate only the issues the user supplies and fixes needed for those issues. Delegate code changes to subagents; your own edits are limited to the ledger and reports. Your job is to dispatch, check independently, and keep the ledger true. The user may override the defaults below.

The run is built to finish unattended. The plan message is its only stop: after it, never wait on the user. Record anything that needs them in the ledger and the pull request, and continue with the work that doesn't. Hard stops in preflight still stop the run. If the user asks to be consulted mid-run, ask them where the steps below would block a ticket, and continue once they answer.

The full suite is the slowest step. Where this skill says to run something in the background, do so if your agent can, and keep working on steps that don't edit the working tree; otherwise run it in the foreground.

## Start the run

The input is an explicit list of issues, which may be split into series, a parent issue, a milestone, or a label:

- Single series: `/implementation-coordinator #12 #13 #14`
- Multiple series: `/implementation-coordinator A: #12 #13; B: #20`
- Grouped instead of listed: `/implementation-coordinator parent #40`, `/implementation-coordinator milestone "v2"`, `/implementation-coordinator label "export"`

1. Resolve the repository, input, series order, and constraints from the request. Use the current Git repository and checked-out branch when unambiguous. Ask one concise question for any missing ticket numbers or unclear order before delegation. Take inputs already available from the request or repository as given.
2. **Preflight.** Check each of these before delegation. When one stops the run, report the exact state and the action needed to resume.
   - `gh auth status` succeeds for the repository's host. A failure stops the run.
   - The repository has a GitHub remote. Without one, stop.
   - The working tree is clean. A dirty tree stops the run.
   - The branch is the requested or inferred work branch. On the default branch with a clean tree, create a work branch with a short name derived from the input (for example `<user>/<series-name>` or `<user>/issue-<first-number>`) and name it in the plan message. Any other branch stops the run. Stay on the work branch for the rest of the run.
   - Push access: `gh repo view --json viewerPermission` returns `ADMIN`, `MAINTAIN` or `WRITE`. Without it, pushing is off for the run; say so in the plan message.
3. **Find the ledger** at `.scratch/implementation-coordinator/<branch>.md` (`/` in the branch name becomes `-`). It must never appear in `git status` or a commit: if `git check-ignore -q .scratch/` fails, append `.scratch/` to `.git/info/exclude`. A ledger with `Run: finished` is renamed to `<branch>.<YYYY-MM-DD>.md` and a new run starts. A ledger with `Run: in progress` is resumed as in [Resume a run](references/resume.md#resume-a-run). Otherwise record the current commit as the base.
4. **Start the baseline.** Tell the user the baseline suite is running, then start the repository's full verification suite on the base in the background: tests, builds, type checks, lint, and integration or end-to-end checks where applicable. If the repository's instructions don't name the commands, take them from its configuration or CI workflow and record what you used. Continue with the next steps while it runs. Record each command, its result and the names of any failing tests in the ledger's baseline table; later failures are compared against it. If the suite cannot run or the build fails at the base, ask in the plan message whether to continue from that baseline or stop.
5. **Expand the input.** For a parent issue, list its sub-issues with `gh api repos/<owner>/<repo>/issues/<n>/sub_issues` and keep the order the API returns, which is the parent's sub-issue order. For a milestone or label, list the open issues with `gh issue list --state open --milestone "<milestone>"` or `gh issue list --state open --label "<label>"`; these have no inherent order, so propose one (blockers first, then ascending issue number) for the user to confirm in the plan message. If an expansion is empty or the API is unavailable, say so and ask for an explicit list instead.
6. **Read the issues.** Read repository instructions and how to run tests. Leave domain, decision, and design documents to the ticket and fix agents, whose briefs ask them to read what is relevant. Fetch every issue with `gh issue view <n> --comments`, confirm its repository, and record its title, acceptance criteria, decisions made in comments, and dependencies. Dependencies come from the issue text and from `gh api repos/<owner>/<repo>/issues/<n>/dependencies/blocked_by` (if that endpoint is unavailable, rely on the text). Note any ambiguity that would change behavior a user sees, tied to its ticket, and flag any ticket that looks done already: its issue is closed, or a commit whose subject ends in `(#<n>)` exists at or before the base (`git log <base> --format=%s`). If GitHub access or issue content is unavailable, report which issue could not be read and why.
7. **Build the ledger** and save it once the baseline has finished. Preserve the supplied order within each series and honor dependencies across series, reordering where a blocker is listed after the ticket it blocks. A blocker in this run is satisfied once its ticket is complete here, even though its issue stays open, or once it is skipped and the user confirmed the skip satisfies that blocker; an open blocker outside the run blocks the ticket.
8. **Plan and ask.** Send one plan message showing the repository, branch, ticket order (with any reordering), blockers, and the baseline result. It asks every question, each naming the ticket it affects:
   - every ambiguity found while reading the issues;
   - for each ticket flagged as done already, whether to skip it and, if it blocks another ticket in the run, whether the skip satisfies that blocker (only a yes counts);
   - the order to confirm, from a milestone or label;
   - a substitute model, if the user named one for subagents and it is unavailable;
   - whether to continue from a failing baseline.

   State the push plan from [Pushing](#pushing), and that this is the run's last stop: after it, a new question blocks only its ticket, and the run continues and lists the question in the pull request. Ask everything here, before any agent starts, and wait for the answers. With no questions, say so and begin delegation without waiting. Record each answer in the ledger's settled answers as soon as you have it, so a resumed run keeps it.

### Ledger format

Keep this exact shape so a later run can resume from it, and update it whenever a status, commit or pull request changes. The ledger is the source of truth: if your context was summarized or you are unsure of the run's state, re-read it and `git log <base>..HEAD` before the next step. Statuses are `pending`, `in progress`, `complete`, `blocked`, and `skipped`. A `blocked` row's Notes start with the blocker (`question:` for an open question) and name any WIP branch. A `skipped` row's Notes say why and, when it blocks another ticket, whether the user said the skip satisfies that blocker. An awaited answer is `open`.

```markdown
# Implementation ledger: <branch>

Repository: <owner>/<repo>
Base: <sha>
PR: <url> | none
Run: in progress | finished

| # | Series | Ticket | Title | Status | Commit | Notes |
|---|--------|--------|-------|--------|--------|-------|
| 1 | A | #12 | Add export endpoint | complete | a1b2c3d | Chose CSV as the default format |
| 2 | A | #13 | Remove legacy export | skipped | | Closed before the run; user said the skip satisfies #14 |
| 3 | A | #14 | Export filters | blocked | | question: include archived rows?; WIP: jane/export-wip-14 |

## Baseline

The full suite at the base, so later failures can be compared with it.

| Command | Result |
|---------|--------|
| npm test | 1 failing: `import > rejects empty file` |

## Settled answers

Questions asked before or during the run, with their answers, so a resumed run reuses them.

| Ticket | Question | Answer |
|--------|----------|--------|
| #12 | CSV or JSON export? | CSV |
| #14 | Include archived rows? Options: yes; no (recommended) | open |

## Fix commits

| Commit | Issues | Finding |
|--------|--------|---------|
```

### Pushing

By default the run pushes the branch with `git push -u origin <branch>` after each completed ticket, opens a draft pull request against the default branch after the first, and finishes it at the end. Pushing is off when push access is missing or the user opted out; if the user asked to push only at the end, only [Finish the pull request](#finish-the-pull-request) pushes. Record the user's choice in the settled answers. Push only commits that passed their checks, so retries and resets never touch pushed history and no force-push is needed. If a push or the draft fails, report the error once, stop pushing, and keep working locally; the final push retries it.

## Implement tickets

Run one implementation or fix agent at a time, and wait for it to finish before the next step. Subagents use the session's model unless the user names one. Tickets marked `skipped` are not dispatched.

For each unblocked ticket:

1. **Dispatch.** Record `HEAD`, mark the ticket `in progress`, and spawn a fresh implementation subagent with [the ticket agent brief](references/ticket-agent-brief.md), filled in from the ledger and the user's constraints.
2. **Park questions.** An agent may return a question instead of a commit when something new comes up. If the ticket or its settled answers already decide it, send that to the same agent (continue it by its agent ID so it keeps its context). Otherwise don't wait for the user: add the question, its options and the agent's recommendation to the settled answers with Answer `open`, and [block the ticket](references/resume.md#block-a-ticket). A question uses no retry. If the user asked to be consulted mid-run, ask them instead, send the answer to the same agent, and record it.
3. **Check completion yourself.** Start the focused-check commands the agent reported in the background, adding your own when they don't clearly cover the changed behavior. While they run, confirm:
   - exactly one new commit exists since the recorded `HEAD`, its subject ends in `(#<n>)`, and the working tree is clean;
   - `git show --stat <sha>` touches only what the ticket needs, and the hunks you read hold no unrelated changes;
   - the agent's acceptance-criteria checklist has every criterion you recorded, each backed by a test or diff location you confirm in `git show <sha> -- <file>`. A missing, unmet or unbacked criterion fails the check;
   - a read of `git show <sha>` along the changed code paths finds no evident bug, such as a wrong condition, an unhandled empty or error case, or a broken caller. One found fails the check.

   Then confirm the focused checks pass; the agent's report of passing tests is a claim until you have seen them pass. Pure configuration or wiring may have nothing independent to test. This is a completion check, not the final code review.
4. **Retry once.** If a check fails, send the specific gaps to the same agent with the fix that matches the failure:
   - **No commit:** finish the ticket and commit it. A returned question is handled by step 2, not here.
   - **Several commits:** squash them into one commit whose subject ends in `(#<n>)`, with `git reset --soft <recorded HEAD>` and a single commit.
   - **Uncommitted or stray changes:** fold the ticket's changes into its commit and discard unrelated ones, explaining each.
   - **Gaps in content or criteria:** fix them and amend the ticket commit.

   Record the resulting hash and run step 3 again.
5. **Block if it still fails.** [Block the ticket](references/resume.md#block-a-ticket), which sets its partial work aside on a WIP branch, then continue with independent tickets.
6. **Record and push.** Mark the ticket complete only after its commit and focused checks pass, update the ledger, and push as in [Pushing](#pushing). After the first complete ticket, if the ledger's `PR:` line is `none` and `gh pr list --head <branch> --state open` finds nothing, open a draft with `gh pr create --draft --base <default branch> --head <branch>`, titled as in [Finish the pull request](#finish-the-pull-request), with a body that says the run is in progress and lists the planned tickets; record its URL on the `PR:` line.
7. **Report.** Keep each progress update to one line per ticket: `✓ #13 Add export endpoint (a1b2c3d), next: #14`, naming the next ticket or, when none remain, the next step. A blocked ticket gets the blocker exactly as found and the action needed to resume.
8. **Checkpoint.** Run the full suite and compare it with the baseline when the last runnable ticket of a series completes and tickets in other series remain, and after every 5 completed tickets since the last checkpoint within a series (the user may change that number). Skip it after the run's last runnable ticket, since verification runs the suite. Report it in one line, for example `✓ checkpoint after series A: suite matches baseline`. Before the next ticket starts, fix any failure the baseline doesn't show as in [Verify the series](#verify-the-series), then rerun the suite. If a failure remains after both attempts, record it as a blocker and dispatch no more tickets, since a broken suite makes later completion checks unreliable: mark the remaining tickets `blocked` with Notes naming the failing check, then go on to verification and finishing; the pull request stays a draft.

## Verify the series

After all runnable tickets are complete or blocked, confirm the working tree is clean and tell the user full verification is starting, since it may take a while. Start the full suite in the background and, at the same time, spawn a Standards, a Spec and a Correctness review agent in parallel with [the review agent briefs](references/review-agent-brief.md); they only read, so they don't disturb the suite. When the suite finishes, compare each failure with the baseline: one the baseline shows is pre-existing, so report it and leave it outside this work; any other failure gets fixed below.

Reviewers can be wrong. Confirm each finding by reading the code at its location and any related tests, instead of reading all of `git diff <base>...HEAD`. Fix confirmed requirement gaps, regressions, crashes or wrong results in the new code, security defects, and documented rule violations. Report debatable code smells as suggestions without fixing them.

Record `HEAD` as the pre-fix `HEAD`, then group the failures and confirmed findings: those that touch the same file or affect the same issue(s) go to one fresh fix agent, unrelated ones to separate agents. Run the agents one at a time with [the fix agent brief](references/fix-agent-brief.md). Check each result like the completion check: one new commit per finding, each subject referencing its affected issue number(s), a clean tree, and every finding resolved with focused checks passing. Preserve the ticket commits, record each fix commit in the fix commits table, and push it like a ticket commit. If the agent cannot reproduce a problem, weigh its evidence and either drop the finding or send a sharper reproduction. Allow at most two fix attempts per failure or finding; a finding that fails as part of a group can be retried on its own. If any fix commits were made, spawn one review of them with [the fix commit review brief](references/review-agent-brief.md#fix-commit-review). It runs once: confirm its findings and fix them with the same fix-agent flow and attempt limit, without reviewing those fixes again, and report any that remain as unresolved. Then rerun the full suite. If evidence shows a new failure is unrelated to the series, such as a flaky test that also fails at the base when rerun, report that evidence and leave it outside this work.

## Finish the pull request

Skip this when push access is missing, the user opted out of pushing, or no ticket is complete; say which in the report.

1. Push with `git push -u origin <branch>`, which also retries a push that failed during the run. If it fails, report the error and skip the rest of this step.
2. Find the existing pull request from the ledger's `PR:` line or `gh pr list --head <branch> --state open`, and update its title and body with `gh pr edit`; with none, create one with `gh pr create --base <default branch> --head <branch>`.
3. Title it after the input: the single ticket's title, or the parent issue, milestone, label or series name.
4. Write the body. If the repository has a pull request template (`.github/pull_request_template.md`, `.github/PULL_REQUEST_TEMPLATE.md`, `PULL_REQUEST_TEMPLATE.md` or `docs/PULL_REQUEST_TEMPLATE.md`), fill in its sections; otherwise use the summary table, full suite results, review findings and suggestions, and risks from the report below. Add one `Closes #<n>` line per complete ticket. List blocked and skipped tickets without closing keywords. A parent issue input gets `Part of #<parent>`, never `Closes`. When blockers remain, add an **Open questions** section: for each blocked ticket, its question with the options and the recommendation, or the blocker, plus its WIP branch. End it with how to resume: answer in a comment on this pull request, then run the same command again on this branch.
5. Mark it ready for review, or keep it a draft and say why in the body when blockers or unresolved review findings remain or the full suite fails. Match its state with `gh pr ready` or `gh pr ready --undo`, or pass `--draft` when creating it.
6. Record the URL on the ledger's `PR:` line, so a resumed run updates this pull request instead of opening another.

## Report the outcome

Set the ledger's `Run:` line to `finished`, or leave it `in progress` when blockers remain so a later run can resume. Leave issues open; the pull request's `Closes` lines close them when it merges. Report in this shape:

```markdown
## Implementation summary: <branch>

| Ticket | Status | Commit | Summary |
|--------|--------|--------|---------|

**Full suite:** each command run and its result, with failures the baseline already showed marked pre-existing.
**Review findings:** each confirmed finding with its fix commit, or "unresolved". Suggestions follow in a separate list.
**Blockers:** each with the specific next action needed to resume: the open question with its options, or the blocker, plus the WIP branch.
**Risks and follow-ups:** anything a reviewer of the branch should know.
**Pull request:** its URL and whether it is ready or a draft and why, or why none was opened.
**Branch state:** final `git status`, and commits since base.
```
