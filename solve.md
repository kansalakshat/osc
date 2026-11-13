# osc: solve item N (Steps 3-9)

Board rules (template, gate, AI policy, CodeRabbit, baseline) are in the board
header; read header + table + item N section (SKILL.md). Don't redo Step 0.

## Step 3: pick up item N

1. Re-check live (things change in hours):
   `gh issue view N -R OWNER/REPO --json assignees,comments,closedByPullRequestsReferences --jq '{a:[.assignees[].login],prs:[.closedByPullRequestsReferences[].url],last:(.comments[-3:]|map("\(.author.login): \(.body[:300])"))}'`
   and confirm still broken on latest `main`. Taken: mark the row, tell the user.
2. **Claim if required.** Draft a short claim comment (what's broken, planned
   fix in one line, ask to be assigned), show it, post only on confirm. Gated on
   assignment: stop here, status `claim pending`.
3. Fresh branch per item:
   `git fetch upstream && git switch -c fix/N-short-name upstream/<default-branch>`

## Step 4: reproduce it yourself

Non-negotiable. Run the failing case, watch it fail. Capture the exact before
output; it goes in the PR body.

## Step 5: fix it, smallest possible

- Grep every caller before editing a shared function.
- Fix the root cause, not the symptom. Route the broken case into machinery
  that already exists.
- Say plainly what's out of scope, and why.

**Match the maintainers.** Read 1-2 small merged maintainer fixes, not more:

```bash
gh pr list -R OWNER/REPO --state merged --limit 10 --json number,title,author,additions,deletions \
  --jq '.[]|"#\(.number) +\(.additions)/-\(.deletions) \(.author.login): \(.title)"'
gh pr diff <smallest bug-fix PR> -R OWNER/REPO | head -150
```

Copy the closest neighbour: the function next to the bug, the test next to the
new test. Most real fixes are 1-10 source lines plus one test; if yours is bigger
than the small merged fixes, cut it down before showing the user.

**Never add:** narrating comments (only a non-obvious *why*, and only if nearby
code has them); docstrings/type hints/logging the file doesn't use;
`try/except` or null checks for impossible cases; a once-used helper or
constant; renames, reformatting, import reorders, drive-by cleanups; long names
where the file uses short ones.

**Tests, never add:** a new test file when one covers the module; a test class
with setup for one assertion (extend an existing parametrize list or method);
assertions beyond proving the fix.

**Delete pass:** read the diff line by line, "would a maintainer remove this?"
If yes, remove it. Re-run Step 6.

## Step 6: prove it

```bash
<runner> <new test>                                   # 1. passes with fix
git stash push <source files> && <runner> <new test>  # 2. source reverted: must FAIL
git stash pop
<runner> <related suites> > $L/related.log 2>&1; tail -30 $L/related.log   # 3.
<linters exactly as .github/workflows/ runs them> > $L/lint.log 2>&1; tail -30 $L/lint.log  # 4.
```

(`L=~/oss-boards/OWNER__REPO.logs`.) Passing with and without the fix proves
nothing. Only failures not in the baseline are yours.

## Step 7: diff actually small?

`git diff --stat upstream/<default-branch>`. Far larger than the change: line
endings.

```bash
python -c "import sys;F=sys.argv[1];b=open(F,'rb').read();open(F,'wb').write(b.replace(b'\r\n',b'\n'))" FILE
```

## Step 8: brief the user in plain language

Short sentences, no jargon they didn't use: what was broken in user terms; what
the fix does and why the simpler fix doesn't work; what was tested incl. the
revert-must-fail check; likely reviewer pushback; out of scope. If they can't
explain it back, they can't answer the reviewer.

## Step 9: commit and draft, then STOP

- Commit in the project's format (`git log --oneline -20`). Django:
  `Fixed #NNNNN -- Past tense description.` Conventional: `fix(scope): ...`.
- Sign-off if required (`git commit -s`; CNCF/LFX usually enforce DCO).
- No AI co-author trailers. Push to the **user's fork**, never upstream.
- PR text in the project's template: keep headings and order, content under the
  right headings, `Fixes #N` where used. Follow any CONTRIBUTING format.
- **Tick every other template checkbox** after making it true (run the
  lint/build/tests it names, add asked screenshots, sign off for DCO, link the
  issue). Can't be made true: leave unticked, tell the user why in one line.
  Never tick an untrue box. **Never touch the AI disclosure** (SKILL.md).
- Text like a busy developer: match merged PRs' length, usually 2-5 plain
  sentences (what broke, why, what changed, how tested). No bullet walls.
- **Stop and show the user.** Open only after they confirm, then set the row
  to `PR open <link>`.
- **CodeRabbit** (if the board says the repo uses it): ~5 min after opening:

  ```bash
  gh pr view <PR> -R OWNER/REPO --json reviews,comments \
    --jq '[.reviews[],.comments[]|select(.author.login|startswith("coderabbitai"))|.body
           |if test("Review skipped";"i") then "skipped" else "ran" end]|unique'
  ```

  `[]` or `["skipped"]`: post `@coderabbitai review` as a PR comment without
  asking. Rate limited: wait the stated time, post once. Never more than once
  per push. Then do Step 10 (`reviews.md`) on its findings.
