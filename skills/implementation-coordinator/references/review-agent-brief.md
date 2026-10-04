# Review agent briefs

Spawn a Standards, a Spec and a Correctness review agent, in parallel, when the full verification suite starts. Fill every placeholder. Reviewers read and report only; they never edit files or commit.

## Shared instructions

> Review the changes on `<branch>` in `<owner>/<repo>` since `<base>`: `git diff <base>...HEAD` and `git log <base>..HEAD`. The series implements these tickets: `<ledger rows>`. Failures already present at `<base>`: `<baseline failures>`; treat those as pre-existing. The coordinator runs the full suite; don't run it yourself.
>
> Read the surrounding code, not just the diff, before judging a change. Do not edit files, stage, or commit.
>
> Report each finding with: file and line, what is wrong, a concrete scenario or evidence showing it, the affected issue number(s), and a severity of **must-fix** (requirement gap, regression, crash or wrong result, security defect, violation of a documented repository rule) or **suggestion** (debatable smell or style preference). Report only what you can point to in the code; say "no findings" if there are none.

## Standards review

> Check the changes against the repository's own instructions, conventions, and documented rules (for example CLAUDE.md, AGENTS.md, CONTRIBUTING, lint and style config, and the patterns in neighboring code). Look for regressions in existing behavior, duplicated logic that should reuse existing code, dead or unreachable code, leftover debugging, and security defects such as injection, unsafe input handling, leaked secrets, or missing authorization checks. Check that new public interfaces (functions, endpoints, CLI flags, configuration) are consistent with the existing ones they sit beside.

## Spec review

> For every ticket, fetch it with `gh issue view <n> --comments` and check each acceptance criterion and each decision in the comments against the code and tests. Mark each criterion met, partially met, or unmet, with evidence. Then check the combined behavior: tickets that conflict, later tickets that undo earlier ones, and edge cases that fall between tickets. Confirm tests assert the specified behavior rather than restating the implementation.

## Correctness review

> Trace the new and changed code paths for logic errors: wrong conditions, off-by-one errors, null, empty and boundary inputs, error and failure paths, resource cleanup, and concurrency or ordering problems. Flag obvious performance problems at realistic input sizes, such as repeated queries or calls inside loops, or quadratic work over unbounded data. Back each finding with a concrete input and the wrong result it produces.

## Fix commit review

After the fix agents finish, spawn one Correctness review agent with the shared instructions and the Correctness review brief, replacing the diff with `git diff <pre-fix HEAD>..HEAD` and the log with `git log <pre-fix HEAD>..HEAD`, so it reviews only the fix commits.
