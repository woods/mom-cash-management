---
name: git-commit
description:
  Use this skill whenever you are about to create a Git commit on the
  user's behalf, or when the user says things like "commit this,"
  "commit that change," "let's commit," or "wrap this up." It governs
  how to survey changes, run pre-commit checks, stage files, propose a
  Tim-Pope-style commit message, wait for explicit approval of the exact
  message, and then run `git commit`. Do not invoke `git commit` without
  going through this skill.
---

# Git commit workflow

This skill defines a deterministic five-phase sequence for making a Git
commit. The goal is to eliminate guesswork about which commands to run,
in what order, with what flags. Follow the phases in order. Do not skip
phases, and do not reorder them.

The single hard rule: **never run `git commit` until the user has
approved the exact commit message you proposed in chat.** "Commit when
you're done" is not pre-approval of a message.

## Conventions used in every command

- Run every command from the **repository root**. If your tool takes a
  working-directory argument, set it once; do not `cd` inside command
  strings.
- Prefix every read-only Git command with `--no-pager` so a pager cannot
  hang an interactive shell: `git --no-pager log`,
  `git --no-pager diff`, `git --no-pager show`, `git --no-pager blame`.
- Never run `sudo` or anything that escalates privileges. If something
  seems to need it, stop and explain.
- Do not use shell substitutions (`$(...)`, backticks, `${VAR}`, process
  substitution). Resolve values yourself or ask.

## Project configuration

Each project should tell this skill two things. If they are not written
down, ask the user once and then record the answer in the project's
`AGENTS.md` (or equivalent) so future sessions inherit it.

1. **Pre-commit check command.** A single command that exits non-zero if
   the working tree is not ready to commit. Examples:
   - `./bin/check`
   - `npm test`
   - `pytest`
   - `bundle exec rspec && bundle exec rubocop`
2. **Formatter command (optional).** A single command that rewrites
   files in place to satisfy the check. Examples:
   - `./bin/format`
   - `npm run format`
   - `ruff format .`

In the phases below, `CHECK` means "the project's pre-commit check
command" and `FORMAT` means "the project's formatter command."

## Phase 1 — Survey the changes

Goal: know exactly what is about to be committed before you touch
anything.

Run these three commands, in this order:

```sh
git status
git --no-pager diff
git --no-pager diff --staged
```

Then look at recent history so the new commit matches the project's tone
and granularity:

```sh
git --no-pager log -n 10 --oneline
```

If anything is surprising — files you don't recognize, half-edited
hunks, leftover debug output, an unexpected branch — **stop and ask the
user** before proceeding. Do not silently `git add` things you cannot
explain.

## Phase 2 — Run pre-commit checks

Goal: do not propose a commit on top of a broken or unformatted tree.

Run the project's check:

```sh
CHECK
```

Interpret the result:

- **Passes.** Continue to Phase 3.
- **Fails on formatting only**, and `FORMAT` is defined. Run `FORMAT`,
  then run `CHECK` again. Do **not** hand-edit files to satisfy a
  formatter. If the second `CHECK` still fails, stop and report.
- **Fails on anything else** (lint errors, failing tests, type errors).
  Stop. Report the failure to the user and ask how to proceed. Do not
  "fix and commit" without confirmation unless the fix is obviously
  trivial and clearly inside the scope the user asked for.

If a step in this phase was skipped on a previous commit, treat that as
a **process problem to fix in the project's instructions or tooling**,
not as a personal lapse to promise to do better next time.

## Phase 3 — Stage the right files

Goal: the staged set is exactly the change you are about to describe —
no more, no less.

Default to **explicit** staging:

```sh
git add path/to/file1 path/to/file2
```

Only use `git add -A` or `git add .` when the user has said so, or when
the survey in Phase 1 made clear that every changed file belongs in this
commit.

Then re-verify:

```sh
git status
git --no-pager diff --staged
```

The staged diff is what the commit message must describe. If the staged
diff covers more than one logical change, split it: stage and commit the
changes in separate commits, each with its own message proposal. Use
`git add -p` when you need to split hunks inside a single file.

## Phase 4 — Propose the commit message

Goal: get the user's approval of the **exact** message text before any
commit happens.

Follow Tim Pope's conventions from
[A note about Git commit messages](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html):

- **Subject line**: capitalized, imperative mood ("Add", "Fix",
  "Refactor", not "Added" / "Adds" / "Adding"), no trailing period, aim
  for ~50 characters and treat 72 as a hard ceiling.
- **Body** (optional, but include one when the change is non-obvious):
  one blank line after the subject, then prose wrapped near 72
  characters, explaining **why** and any context a future reader will
  need. Separate paragraphs with blank lines. Bullet lists with `-` are
  fine.
- Match the granularity and tone of recent commits seen in Phase 1.

Writing the message is a **two-pass process**. Drafting and auditing are
different jobs; do not collapse them into one. The rule version of this
used to read like advice you could follow while composing, and that does
not work — in writer mode you will rationalize that the draft already
passes the tests. Treat the audit as a gate that has to be cleared on
its own, with the staged diff in front of you.

### Draft as a tour guide

Imagine the commit message as a tour guide standing next to the diff.
The diff is what visitors came to look at; the message tells them what
the authors experienced and why they crafted the change the way they
did. Across a series of commits, the messages become a readable log of
how the project's thinking evolved — useful on their own, without having
to reconstruct context from patches.

Draft toward that voice. Don't audit yet — write freely, capturing the
reason, the conversation, the constraint, the design choice, the
alternative considered and rejected, the follow-up the reader should
know is coming. If some sentences also describe what the diff shows, let
them through for now. The next pass will trim.

### Audit against the diff

Once the draft is on the page, switch hats from author to editor. Pull
up the staged diff again — actually re-read it, do not work from memory
— and go through the draft sentence by sentence. For each sentence, ask:
would a reader who is already looking at the diff learn anything new
from this sentence? If no, cut it.

A worked example. "Added a 'What we mean by product' section" goes; the
diff already shows the section appearing. "Added it because newcomers
were confused that we're not a SaaS" stays; that's the part a reader
can't see.

If the body is empty after the audit, leave it empty. A subject-only
commit is fine when there is nothing to say beyond the diff.

Do not show the message in chat until this audit is done. Showing a
draft before the audit skips the gate.

### Displaying the message in chat

Show the message to the user as a **fenced code block tagged `text`**,
containing only the literal bytes that will be committed — no leading
prose inside the fence, no quote marks around it, no `>` quoting, and no
trailing `EOF` markers.

A `text` fence preserves:

- Exact line breaks between subject, blank line, and body.
- Trailing/internal whitespace.
- Markdown-significant characters (`#`, `*`, `_`, backticks) without
  them being interpreted as headings, emphasis, or inline code.

The rendered result should look like this in chat:

```text
Fix off-by-one in pagination cursor

The cursor was advancing past the last row when the page size evenly
divided the result count, which caused the final page to render empty.
Clamp the cursor to len-1 before the comparison.
```

Do not put a heading like "Proposed commit message:" inside the fence;
put any framing prose outside it. If your host chat environment requires
a different fence convention (for example, a path-tagged fence), follow
that convention but keep the same rule: only the literal commit bytes go
inside.

After the fence, offer the user a **two-letter reply menu** and then
**stop**. The menu has exactly two letters because there are exactly two
actions that need a shortcut — commit, or don't commit. Everything else
is a conversation, and conversations don't need a hotkey.

- **y** — approve as-is and commit.
- **n** — don't commit. End the workflow and return control.

Format the prompt on its own line, right after the fenced message:

> Reply **y** to commit, **n** to abandon, or tell me what to change.

Do not run `git commit` yet. Do not run it even if the user previously
said "go ahead and commit when ready" — that is permission for the
workflow, not approval of this specific message.

Interpreting the reply:

- **Affirmative** — `y`, `Y`, `yes`, `looks good`, `lgtm`, `ship it`, or
  any clear approval. Proceed to Phase 5.
- **Negative** — `n`, `N`, `no`, `cancel`, `stop`, or any clear refusal.
  Stop the workflow. Do not commit. Acknowledge in chat and return
  control to the user.
- **Edits in prose** — "make the subject shorter," "drop the second
  paragraph," "mention the bug number," etc. Apply the changes,
  re-propose with a fresh fence and menu, stay in Phase 4.
- **A full replacement message** — multi-line text that looks like a
  commit message. Treat as approval of _that_ message. Use it byte for
  byte. Proceed to Phase 5.
- **Silence, a clarifying question, or anything ambiguous** — stay in
  Phase 4. Ask a question back if you need to; do not assume approval.

The menu is a convenience, not a constraint. The user does not need to
type a letter at all — prose works fine. The point of the two letters is
that they are the _fastest possible_ way to commit or abandon when no
discussion is needed.

## Phase 5 — Commit

Goal: create the commit with the approved message, then confirm the
result.

There are two methods. Use the first that applies.

### Method 0: subject-only

For a single-line subject with no body and no special characters (no
single quotes, backticks, or `$`):

```sh
git commit -m "Subject line approved by the user"
```

Then skip to the confirm step below — no cleanup needed.

### Method 1: heredoc piped to `git commit -F -`

For any message with a body, or one that contains characters that would
need shell quoting:

```sh
git commit -F - <<'COMMIT_MSG_EOF_MARKER'
Subject line approved by the user

Body paragraph one, wrapped near 72 columns, explaining why this
change exists and anything a future reader will need.
COMMIT_MSG_EOF_MARKER
```

Requirements:

- The delimiter **must** be single-quoted (`<<'COMMIT_MSG_EOF_MARKER'`).
  That disables variable expansion and command substitution, so `$`,
  backticks, and `\` pass through literally.
- The delimiter **must** be unusual enough that it cannot appear in the
  message body. `COMMIT_MSG_EOF_MARKER` is fine; `EOF` is not.
- Do not indent the heredoc body and do not use `<<-`; those strip
  leading whitespace.

### Confirm

After committing by either method:

```sh
git --no-pager show --stat HEAD
```

Check that the subject and body match what the user approved exactly. If
they do not, amend:

```sh
git commit --amend -F - <<'COMMIT_MSG_EOF_MARKER'
Corrected message here.
COMMIT_MSG_EOF_MARKER
```

### Report

Tell the user: the short SHA, the subject line, and the list of files
from the `--stat` output. That is the end of the workflow.

## What to do when something goes wrong

- **You committed without approval.** Tell the user immediately. Offer
  `git reset --soft HEAD~1` to undo the commit while keeping the staged
  changes, so you can re-enter Phase 4 with a proposed message.
- **You committed the wrong files.** Tell the user. Depending on what
  they want, either `git reset --soft HEAD~1` and re-stage, or make a
  follow-up commit.
- **Pre-commit checks fail after staging.** Unstage with
  `git restore --staged path` if needed, fix the underlying issue, and
  restart from Phase 2.
- **You already pushed.** Do not rewrite shared history without explicit
  instruction. Propose a follow-up commit instead.

## Quick reference

| Phase      | Command(s)                                                                  |
| ---------- | --------------------------------------------------------------------------- |
| 1. Survey  | `git status` · `git --no-pager diff` · `git --no-pager diff --staged`       |
| 1. History | `git --no-pager log -n 10 --oneline`                                        |
| 2. Check   | `CHECK` (then `FORMAT` + `CHECK` if formatting failed)                      |
| 3. Stage   | `git add <paths>` then `git --no-pager diff --staged`                       |
| 4. Propose | Post message in chat + `y`/`n` menu. **Stop.** Wait for reply.              |
| 5. Method  | Subject-only: `git commit -m "…"` · With body: heredoc to `git commit -F -` |
| 5. Confirm | `git --no-pager show --stat HEAD`                                           |
