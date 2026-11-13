---
name: osc
description: "Give it a GitHub repo URL and it builds a Problem Board of every workable problem (issues, failing tests, lint errors, broken docs, fuzz bugs, abandoned PRs) with root cause, fix, test plan and size; then 'solve N' takes an item to a verified PR, paused for confirmation. Tuned for GSoC, LFX, MLH, Summer of Bitcoin, ESoC, C4GT. Use when the user gives a repo link and wants to contribute or prep for a mentorship program."
triggers:
  - "\\bosc\\b"
  - "oss.?contribute"
  - "contribute.?to"
  - "find.?(me.?)?(an.?)?issues?"
  - "open.?source.?contribution"
  - "draft.?(a.?)?pr"
  - "problem.?board"
  - "solve.?#?\\d+"
  - "gsoc|lfx|mlh.?fellowship|summer.?of.?bitcoin|esoc|c4gt|dmp"
---

# osc

Repo URL in, numbered Problem Board out. Per picked item: a verified PR on the
user's fork, paused for confirmation.

## Commands: read only the file the command needs

Files sit next to this one (`~/.claude/skills/osc/`). Don't read a file the
current command doesn't name. Already read this session? Don't re-read.

| User says | Read | Do |
|---|---|---|
| `<repo url>` | `board.md` | Steps 0-2: recon, scan, show board |
| `solve N` (any order) | `solve.md` | Steps 3-9 for item N |
| `verify N` / `more` | `board.md` "Deep triage" only | Triage item N / next ~10 unverified, update rows |
| `status` | none | Board table; refresh each `PR open` row with `gh pr view <PR> --json state,reviewDecision` |
| `reviews N` | `reviews.md` | Step 10 |
| `proposal` | none | Step 11 below |

## Board file = memory

`~/oss-boards/OWNER__REPO.md`. Always read it before acting on a number; never
trust a number from chat. Read only what you need, not the whole file:

```bash
B=~/oss-boards/OWNER__REPO.md
sed -n '1,/^## 1\./p' "$B" | head -n -1                  # header + table
sed -n "/^## $N\./,/^## /p" "$B" | head -n -1             # item N section
```

## Hard rules (all commands)

- Nothing is pushed, opened, claimed, or commented without the user saying so.
  Exceptions: fixes pushed to an already open PR (Step 10), and
  `@coderabbitai review` when the bot didn't run (Step 9).
- **Messages to people** (GitHub, Discord, Slack, email, list): put the
  ready-to-send draft in chat; never send it yourself. Follow the project's
  format. Write like a busy developer: short, plain, no "This PR aims to",
  "comprehensive", "robust", "seamless", "leverage", "ensure", no emoji.
- **Disclosure: the user always handles AI disclosure manually.** Never tick an
  AI checkbox, pick an option, or write a disclosure statement, even if marked
  mandatory. Leave it as the template has it, surface the exact policy wording,
  remind the user before the PR opens. No policy: say nothing. Never add AI
  co-author trailers; the user is sole author. Maintainers close unverified AI
  work on sight ("Closing this as ai slop"); verification (Steps 4, 6) and the
  user being able to explain it (Step 8) are what prevent that.
- **Pacing:** max 2 open PRs per repo. Merged beats opened; three small merged
  fixes in one org beat thirty drive-bys.
- **Keep tool output small.** Long-running output (tests, lint, builds, fuzz)
  goes to a log file; read `tail -40` or `grep` it, never dump it. Use `--json`
  + `--jq` on `gh` calls; never `--comments` or raw bodies of bot walkthroughs.
- **GitHub rate limits:** use `gh issue list`/`gh pr list` (GraphQL), not
  `gh api search/issues --paginate`. On a 403 rate limit, `gh api rate_limit`
  and wait for the reset; never retry in a loop.

## Step 11: application kit (`proposal`)

From the board's `merged` and `PR open` rows:

- Contributions list: PR link, one-line user-facing impact, status.
- Areas of the codebase touched, mapped to the program idea being applied for.
- A draft paragraph in the user's voice, which they rewrite. Programs screen
  for people who understand their own work.
