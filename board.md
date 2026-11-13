# osc: build the board (Steps 0-2)

## Step 0: recon the project's rules (never skip)

```bash
gh repo view OWNER/REPO --json url,description,stargazerCount,pushedAt,isArchived,defaultBranchRef,primaryLanguage
for f in CONTRIBUTING.md .github/CONTRIBUTING.md .github/pull_request_template.md \
         .github/PULL_REQUEST_TEMPLATE.md docs/CONTRIBUTING.md AI_POLICY.md .coderabbit.yaml; do
  curl -s -o /dev/null -w "%{http_code} $f\n" \
    "https://raw.githubusercontent.com/OWNER/REPO/HEAD/$f"
done
gh label list -R OWNER/REPO -L 300 --json name --jq '.[].name' \
  | grep -iE 'gsoc|lfx|mentor|c4gt|dmp|esoc|bitcoin|mlh|good.?first|help.?wanted|bug|docs'
```

Establish and report:

1. **Archived or dormant?** `isArchived`, `pushedAt` older than ~6 months, or
   open PRs unreviewed for months: stop. Nothing will merge.
2. **Where do issues live?** Django uses Trac (GitHub is PR-only). Blender's
   GitHub is a read-only mirror. Some use GitLab, Jira, or a mailing list.
3. **Gate?** Django needs the ticket `Accepted` and assigned; a bot closes
   non-compliant PRs in seconds. Many program repos (C4GT, most LFX) require
   assignment before work. Note the exact rule.
4. **PR template:** read it verbatim. Also check `.github/PULL_REQUEST_TEMPLATE/`
   and, if none, the org-level `OWNER/.github` repo. Note any format CONTRIBUTING
   sets for PR text or comments. Note CodeRabbit use (`.coderabbit.yaml` above,
   or `coderabbitai` reviews on recent PRs).
5. **AI policy:** grep CONTRIBUTING, template, AI_POLICY for
   `AI|LLM|generated|Copilot` (grep the fetched text; don't paste whole files).
6. **Program fit:** program labels from the grep, then official listings below.
   One line per matching program with its current window. Fetch dates live.

| Program | Listings | Selection weighs | Notes |
|---|---|---|---|
| GSoC | summerofcode.withgoogle.com (orgs ~Feb, apps ~late Mar) | Early mentor contact, merged PRs in org, concrete proposal | Past-year orgs usually return; start months early |
| LFX | mentorship.lfx.linuxfoundation.org; `github.com/cncf/mentoring` | Prerequisite tasks, project cover letter (= proposal), prior PRs | 3 terms/yr; DCO common |
| MLH Fellowship | fellowship.mlh.io | Interview walking through your code | OSS track not every batch |
| Summer of Bitcoin | summerofbitcoin.org | Portfolio pre-screen, challenge round, security thinking, responsible AI use | University students; Bitcoin projects |
| ESoC | esoc.dev, `github.com/european-summer-of-code` | Each card's own "how to apply" | Europe-based projects, global eligibility |
| C4GT DMP | codeforgoodtech.in | Contributions to listed project, fit with 2-3 orgs | India-focused DPGs; apps ~Apr; claim usually required |

## Step 1: set up locally (once per repo)

```bash
gh repo fork OWNER/REPO --clone=false            # only if not already forked
git clone https://github.com/USER/REPO && cd REPO
git config core.autocrlf false && git config core.eol lf   # BEFORE any edit (Windows)
git remote add upstream https://github.com/OWNER/REPO
```

Install dev deps per CONTRIBUTING, then run tests and linters once on untouched
`main`, output to logs:

```bash
L=~/oss-boards/OWNER__REPO.logs; mkdir -p $L
<test runner, quiet flags> > $L/baseline-tests.log 2>&1; tail -40 $L/baseline-tests.log
<linters as CI runs them>  > $L/baseline-lint.log  2>&1; tail -40 $L/baseline-lint.log
```

Record the baseline: which tests already fail, which lint errors exist.

## Step 2: build the Problem Board

Scan every source. List everything that survives the quick filter. Deep-triage
the top ~15; mark the rest `unverified` (a guess, never presented as confirmed).

### Source A: filed issues

```bash
gh issue list -R OWNER/REPO --state open --limit 500 \
  --search 'no:assignee -linked:pr sort:updated-desc' \
  --json number,title,comments,updatedAt,labels \
  --jq '.[]|"#\(.number)\t\(.comments|length)c\t\(.updatedAt[:10])\t\([.labels[].name]|join(","))\t\(.title)"'
gh issue list -R OWNER/REPO --state open --label "LABEL" --limit 100 \
  --json number,title,assignees,comments \
  --jq '.[]|"#\(.number)\t\(.comments|length)c\t\([.assignees[].login]|join(","))\t\(.title)"'
```

`good first issue` goes in days in mature projects; ordinary bug reports are
where free work is.

### Source B: problems nobody filed (higher value: no queue)

Run every check. Code reading covers **every** source file in the app (large
repo: every file of the modules in the program's idea area), not a sample. A
short board must mean a clean repo, not an early stop. Record coverage.

- **Failing tests on main** from the baseline log (skip flaky/network).
- **Configured linters/type checkers** only (`pyproject.toml`, `.eslintrc`,
  `golangci.yml`, clippy): only what the project's own config flags.
- **Deprecation warnings:** `grep -i deprecat $L/baseline-tests.log | sort | uniq -c | sort -rn | head -20`.
- **Docs that lie:** README/docs examples that don't run, broken links
  (`lychee` or `curl -I`), documented flags that don't exist.
- **Fuzz the public API:** Hypothesis, fast-check, `go test -fuzz`,
  proptest/cargo-fuzz. ~1 real bug per 30 candidates. Logs to `$L`.
- **`TODO`/`FIXME`/`XXX`** that reference a real bug: `grep -rnE 'TODO|FIXME|XXX'`, not file reads.

**File walk via subagents.** Don't read source files in the main context. List
files (`git ls-files` filtered to source), split into batches of ~15, and send
batches in parallel to `Explore` agents, one message. Brief each: "Read every
file fully: <paths>. Report only likely real bugs (wrong logic, unhandled
input on a public path, mismatch between code and its docs/docstring) as
`file:line | symptom | why`, max 10, no style issues, no summaries. End with
`read: N/N files`." Sum the `read:` counts for the Coverage line. Then open
only the flagged lines yourself to confirm.

### Source C: abandoned PRs

```bash
gh pr list -R OWNER/REPO --state open --limit 100 \
  --search 'updated:<YYYY-MM-DD sort:updated-asc' \
  --json number,title,author,updatedAt,reviewDecision \
  --jq '.[]|"#\(.number) \(.title) by \(.author.login) updated \(.updatedAt[:10]) \(.reviewDecision)"'
```

Date ~60 days back. Maintainer asked for changes, author silent: adoptable.
**Ask on the PR first**, credit the original author with a `Co-authored-by:`
trailer (a human, so allowed).

### Quick filter (drop; list under "Skipped" with reason)

- Assigned, or someone said "I'm working on this".
- Linked PR, or a "no patch" flag contradicted by an open PR.
- Old + many comments: unresolved design argument.
- Needs DB, GPU, cloud account, or OS you can't run locally.
- Feature request without maintainer agreement on design.
- Already fixed on `main` (`git log -S "<code>"` and by running it).

### Deep triage (per item)

1. Read every comment, compactly:
   `gh issue view N -R OWNER/REPO --json body,comments --jq '.body, (.comments[]|"\(.author.login): \(.body)")'`
2. Reproduce on current `main` via the public entry point a user touches.
   Retry with *correctly typed* input; a dict where a string belongs proves nothing.
3. Root cause (`file:line`); grep every caller of that code.
4. Proposed fix: fixes the root cause first, then fewest lines and files, with
   estimated size. Say why the obvious fix is wrong if it is.
5. Test plan: which test file, what it asserts.
6. Size: S (<30 lines, one file), M (a few files), L (design needed; say so).

### Board format

Save to `~/oss-boards/OWNER__REPO.md`; show the user the header + table +
Skipped (not every item section; they can ask).

```markdown
# Problem Board: OWNER/REPO
Scanned: YYYY-MM-DD at <main sha> · Programs: GSoC 2027 (org), LFX (label `mentorship`)
Rules: assignment required before PR · AI policy: <none | quote> · Baseline: 3 tests failing on main
Coverage: A 41 open → 9 free · B 62/62 files read, lint, tests, docs links, fuzz (api.py) → 7 · C 3 stale PRs → 1 · not scanned: <what, why>

| # | Problem | Source | Size | Repro | Status |
|---|---------|--------|------|-------|--------|
| 1 | `first` filter crashes on dict | #123 | S | ✅ reproduced | free |
| 3 | Timeout ignored in retry loop | #456 | M | ⏳ unverified | free |

## 1. `first` filter crashes on dict
- Source: <issue link> (2 comments, maintainer confirmed)
- Reproduce: `<exact command>` → `<exact error>`
- Root cause: `src/filters.py:88` indexes `value[0]`; dicts aren't indexable
- Proposed fix (~1 line, 1 file): route through existing `_first_item()` helper
- Why not the obvious fix: `try/except KeyError` hides a real KeyError on empty lists
- Test plan: `tests/test_filters.py::test_first_dict`, asserts first key returned
- Program value: touches the area of the GSoC idea "X"

## Skipped
- #789: assigned to @someone on 2026-08-30
```

Order: reproduced before unverified, then S/M/L, then program idea area first.
Status values: `free`, `claim pending`, `in progress`, `PR drafted`,
`PR open <link>`, `merged`, `closed`, `taken by someone else`.

Then stop and wait for the user to pick.
