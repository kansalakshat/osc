# osc

An agent skill for contributing to open source. Give it a GitHub repo URL and it
builds a numbered Problem Board: filed issues, failing tests, lint and type
errors, broken docs, fuzz-found bugs and abandoned PRs. Each checked item gets a
root cause, proposed fix, test plan and size. Pick an item by number and it
prepares a tested PR on your fork, then waits for your go-ahead.

Tuned for GSoC, LFX Mentorship, MLH Fellowship, Summer of Bitcoin, European
Summer of Code and C4GT.

## Install

Needs `git`, Python and an authenticated [GitHub CLI](https://cli.github.com).

```bash
git clone https://github.com/kansalakshat/osc ~/.claude/skills/osc
```

## Usage

Give it a repo URL, then use these commands:

| Command | What it does |
|---|---|
| `<repo url>` | Checks the project's rules, scans it, shows the board |
| `solve N` | Takes item N from reproduction to a drafted PR |
| `verify N` | Deep-triages an unverified item |
| `more` | Deep-triages the next batch of unverified items |
| `status` | Re-reads the board and refreshes PR states |
| `reviews N` | Fixes bot findings, CI failures and review comments on item N's PR |
| `proposal` | Builds a contribution summary for a program application |

Boards are saved to `~/oss-boards/OWNER__REPO.md`, so work carries across
sessions.

## What it does without asking

Nothing is pushed, opened, claimed or commented unless you confirm it. The one
exception: fixes to a PR that is already open are pushed straight to your fork
branch. If history must be rewritten, that push uses `--force-with-lease`.

It never fills in an AI disclosure. If a project requires one, it shows you the
policy and leaves that part to you.

It keeps at most 2 PRs open per repo.

## License

[MIT](LICENSE)
