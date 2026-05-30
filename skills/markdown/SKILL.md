---
name: markdown
description:
  Use this skill whenever you are editing, creating, or reviewing
  Markdown files in a project, or setting up Markdown tooling in a new
  project. It defines the toolchain (Prettier for formatting,
  markdownlint-cli2 for structure), the two-script workflow
  (`./bin/format` and `./bin/check`), the writing discipline that goes
  with automated formatting (long single-line paragraphs, do not
  hand-format, do not undo the formatter), and the structural and link
  conventions that make Markdown readable on first skim.
---

# Markdown workflow

Markdown in this project is **written loosely and normalized by tools**.
The author's job is the words and the structure; the tools own line
width, list markers, blank lines, table alignment, fence style, and
emphasis style. This skill is the contract that makes that division of
labor work.

The single hard rule: **after editing any `.md` file, run
`./bin/format`, then `./bin/check`, before you hand control back to the
user or propose a commit.** Anything the formatter wrote is, by
definition, the correct output — do not reopen the file to "fix" what it
did.

## The toolchain

Two tools, one each for two different jobs:

- **[Prettier](https://prettier.io/)** — formatter. Rewrites Markdown
  files in place to a consistent shape: wraps prose to a configured
  width, normalizes list markers and indentation, collapses blank lines,
  aligns tables, picks a fence style. Configured by `.prettierrc.yaml`.
- **[markdownlint-cli2](https://github.com/DavidAnson/markdownlint-cli2)**
  — linter. Checks _structure_, not prose. Catches things like skipped
  heading levels, duplicate headings, broken list nesting, and (by
  default) missing language tags on code fences. Configured by
  `.markdownlint.yaml` for rule selection and `.markdownlint-cli2.yaml`
  for tool options like file ignores.

These are not interchangeable. Prettier does not catch structural
mistakes; markdownlint does not reflow prose. The project runs both.

Both `./bin/format` and `./bin/check` assume `prettier` and
`markdownlint-cli2` are on `PATH`. How you get them there depends on
where the agent is running. **Detect the environment first; do not
guess.**

## Installing the tools

### Step 1: Detect the environment

Run these checks, in this order, before suggesting any install command.
They are independent commands so you do not need shell substitution.

1. **OS check.** Run `uname -s`. If it prints `Darwin`, you are on macOS
   (or a macOS-hosted VM). If it prints `Linux`, you are on Linux — most
   likely inside a container in this context.
2. **Container check.** Test whether `/.dockerenv` exists:
   `test -f /.dockerenv && echo container || echo host`. If it prints
   `container`, you are inside a Docker/OCI container.
3. **Devcontainer / Codespaces check.** Run
   `printenv REMOTE_CONTAINERS CODESPACES DEVCONTAINER` (any one of
   those being set is a strong signal that you are inside a VS Code
   devcontainer or GitHub Codespace specifically, not just a generic
   container).
4. **Project devcontainer check.** Test whether
   `.devcontainer/devcontainer.json` exists in the repo:
   `test -f .devcontainer/devcontainer.json && echo yes || echo no`. If
   `yes`, the project supports a devcontainer workflow regardless of
   where you are right now.

Combine the answers into one of three situations and follow the matching
branch below.

### Branch A: macOS host, no project devcontainer

Signals: `uname -s` is `Darwin`, `/.dockerenv` is absent, no
`.devcontainer/` in the repo.

Install with Homebrew:

```sh
brew install prettier markdownlint-cli2
```

Verify:

```sh
prettier --version
markdownlint-cli2 --version
```

Document the command in the project's `README.md` under a "Setup"
heading so the next contributor finds it.

### Branch B: Inside a devcontainer (or any Linux container)

Signals: `/.dockerenv` exists, or one of `REMOTE_CONTAINERS` /
`CODESPACES` / `DEVCONTAINER` is set, or `uname -s` is `Linux` in a
context where the agent runs inside the project's container.

**Do not `apt-get install` prettier or markdownlint-cli2.** They are npm
packages and the Debian/Ubuntu base images devcontainers use do not ship
recent versions. Use npm, and make the install part of the devcontainer
definition so it survives a rebuild.

Edit `.devcontainer/devcontainer.json` so it has both of these keys
(merge with anything already present; do not overwrite):

```json
{
  "features": {
    "ghcr.io/devcontainers/features/node:1": {}
  },
  "postCreateCommand": "npm install -g prettier markdownlint-cli2"
}
```

Notes on merging:

- If `features` already exists, add the Node entry alongside the others;
  do not replace the block.
- If `postCreateCommand` already exists as a string, change it to an
  array:
  `["<existing command>", "npm install -g prettier markdownlint-cli2"]`.
  If it is already an array, append the npm command. If it is an object
  (named commands), add a new entry like
  `"install-markdown-tools": "npm install -g prettier markdownlint-cli2"`.
- If the devcontainer uses a `Dockerfile` instead of features, add a
  `RUN npm install -g prettier markdownlint-cli2` line after Node is
  installed.

After editing, the developer needs to **rebuild the container** for the
install to take effect (in VS Code: "Dev Containers: Rebuild
Container"). Tell them this explicitly; do not assume they know.

If they also need the tools available in the **current** session before
a rebuild (for example, to run `./bin/format` right now), run:

```sh
npm install -g prettier markdownlint-cli2
```

This is a one-time per-session convenience; the devcontainer.json change
is what makes it durable.

### Branch C: macOS host with a project devcontainer

Signals: `uname -s` is `Darwin`, `/.dockerenv` is absent, but
`.devcontainer/devcontainer.json` exists.

Ask the developer one question: **"Do you work on this repo from your
Mac host, from inside the devcontainer, or both?"** Then:

- **Host only** — follow Branch A.
- **Container only** — follow Branch B (you'll edit
  `.devcontainer/devcontainer.json` from the host, which is fine).
- **Both** — do Branch A _and_ Branch B, so the tooling exists in either
  workflow.

Do not silently pick one. The choice changes what gets committed to the
repo.

### A note on agent execution context

The agent itself may run on the host (e.g. an editor extension on macOS
that shells out to the local terminal) or inside the container (e.g. a
VS Code agent attached to a running devcontainer). The detection steps
above describe **where the agent's shell commands actually execute**,
which is the only environment that matters for `./bin/format` and
`./bin/check` to work. Do not reason about it from outside that — run
the checks.

## Project setup (one-time)

A project using this skill should have four files. They are short enough
to reproduce in full here so you can drop them into a new project
verbatim.

### `bin/format`

```sh
#!/usr/bin/env bash
# Rewrite all Markdown files using Prettier. See AGENTS.md.
set -euo pipefail
prettier --write --log-level warn "**/*.md"
```

Make it executable: `chmod +x bin/format`.

### `bin/check`

```sh
#!/usr/bin/env bash
# Markdown checks: Prettier (formatting) + markdownlint-cli2 (structure).
# Run bin/format to fix formatting failures. See AGENTS.md.
set -euo pipefail
prettier --check "**/*.md"
markdownlint-cli2 "**/*.md"
```

Make it executable: `chmod +x bin/check`.

### `.prettierrc.yaml`

```yaml
# Prettier reads this from the repo root.
# https://prettier.io/docs/en/options.html
# We use Prettier only for Markdown.

# Reflow prose to printWidth (rather than preserving hand-wrapping).
proseWrap: always

# Target line length. 72 matches book/article reading widths and Git
# commit body conventions, and fits comfortably in a narrow editor
# pane next to an agent window. Other defensible values: 80, 100.
printWidth: 72
```

`proseWrap: always` is the essential setting. Without it, Prettier
preserves the author's line breaks, which defeats the "write long lines,
let the formatter wrap" discipline below.

### `.markdownlint.yaml`

```yaml
# markdownlint-cli2 reads this from the repo root.
# https://github.com/DavidAnson/markdownlint/blob/main/doc/Rules.md
# Focus: valid, consistent Markdown structure (not prose style or line length).

default: true

# Long lines and URL breaks — Prettier owns wrapping, not the linter.
MD013: false

# Allow inline HTML if we need it rarely.
MD033: false

# Some files may start with a lead-in paragraph before the first heading.
MD041: false
```

The three disables are deliberate, not laziness:

- **MD013** (line length) — Prettier owns wrapping. If you leave MD013
  on, the two tools fight over the same concern.
- **MD033** (no inline HTML) — sometimes needed for
  `<!-- markdownlint-disable -->` comments, embedded `<br>`, or HTML
  comments for review notes.
- **MD041** (first line must be a top-level heading) — some files are
  notes or fragments that don't lead with `#`.

Everything else stays on. In particular, **MD040** (code fences should
have a language tag) is left on — that's why the git-commit skill
specifies `text` as the fence tag.

### `.markdownlint-cli2.yaml`

```yaml
# markdownlint-cli2 tool-specific options.
# Rule configuration lives in .markdownlint.yaml.
# https://github.com/DavidAnson/markdownlint-cli2#configuration

# Files preserved verbatim from outside sources (e.g. raw pastes from
# external docs) are exempted from structural rules. We keep them as
# faithful records of the original; we do not restructure them in
# place. If we want a linted version, we create a new file rather than
# rewriting the source.
ignores:
  - "reference/some-verbatim-paste.md"
```

Use `ignores` for any file that is a faithful copy of external content.
Do not silently restructure such files to satisfy the linter — that
destroys their value as a source-of-truth record. If you need a clean
version, create a new file alongside it.

## Continuous integration

The same `./bin/check` that runs locally before a commit should also run
in CI on every push and pull request. This is not redundant with the
local check — it catches three things the local check cannot:

- Commits that slipped past the local check (someone bypassed
  `./bin/check`, or never installed the tools).
- Contributions from people who don't yet have the tools on their
  machine.
- Drift over time when a rule changes or a formatter version updates and
  a previously-clean file is now stale.

The CI install path uses **npm**, matching
[Branch B](#branch-b-inside-a-devcontainer-or-any-linux-container) of
the install instructions. Do not use `apt-get` in CI for the same reason
— outdated versions.

### GitHub Actions

Drop this at `.github/workflows/check.yml`:

```yaml
name: Check

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  docs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install Markdown tools
        run: npm install -g prettier markdownlint-cli2
      - name: Markdown hygiene
        run: ./bin/check
```

Notes on this workflow:

- **`workflow_dispatch`** lets you re-run the check manually from the
  Actions tab without pushing a commit. Useful when you've changed a
  rule in `.markdownlint.yaml` and want to verify against the current
  `main`.
- **No Node setup step is needed.** GitHub-hosted `ubuntu-latest`
  runners ship with Node and npm already; adding `actions/setup-node` is
  wasted time for this workload.
- **Pin actions to a major version** (`@v4`), not `@main`. `@main` will
  eventually break you on an unrelated day.
- **The job is named `docs`** because in a docs-only repo that's
  accurate. Rename it (`markdown`, `lint`, `check`) if your project does
  more than docs and you want the CI dashboard to be specific.
- **No write permissions.** The default `contents: read` is enough for
  the check job. Only grant `contents: write` if a separate job in the
  same workflow needs to publish a release or push commits.

### Other CI systems

The pattern is the same on any CI provider:

1. Check out the repo.
2. `npm install -g prettier markdownlint-cli2`.
3. `./bin/check`.

Translate the three steps into GitLab CI, CircleCI, Buildkite, Jenkins,
etc. as needed. The important constraint is that **CI runs the same
`./bin/check` script that local developers run**, not a reimplementation
of the check in YAML. If CI and local diverge, you get "works on my
machine" failures, and one of them gets ignored.

### Branch protection

If the repo enforces branch protection on `main`, add the CI check job
as a required status check. That makes the local-and-CI loop genuinely
binding: a PR cannot merge until `./bin/check` passes in CI, which means
it cannot merge with unformatted or structurally-broken Markdown.

## The workflow

The loop after any Markdown edit is exactly two steps:

1. **`./bin/format`** — rewrites the files you touched (and any others
   that were out of shape). Run this before doing anything else with the
   file. Do not preview the formatted output and then re-edit it to
   taste; that fight is unwinnable.
2. **`./bin/check`** — verifies that (a) every file is Prettier-clean
   and (b) every file passes markdownlint. Exits non-zero if anything is
   wrong.

If `./bin/check` fails, follow the rules in the next section. Do not
propose a commit while it is failing.

## Writing discipline

These are the habits that make automated formatting tolerable.

### Write long lines

Output each paragraph as a **single line**. Prettier wraps it to the
configured `printWidth`. Do not target a specific column yourself — you
do not need to know what it is, and guessing wrong wastes diff space and
review time.

The only reason to break a paragraph into multiple lines manually is
when those lines are semantically separate (list items, table rows,
code, blockquoted lines). Sentences inside the same paragraph share a
line.

Do **not** break URLs to wrap. Prettier will leave a long URL alone
rather than wrapping it; that is correct.

### Do not hand-format

Do not insert manual line breaks to make a paragraph "look right." Do
not align table columns by hand. Do not pick between `-` and `*` for
list markers. Do not pick between `_emphasis_` and `*emphasis*`. The
formatter has opinions about all of these; let it win.

### Do not undo the formatter

If `./bin/format` rewrote your file, that is the correct output for this
project. Do not reopen the file and revert any of its choices. If you
genuinely disagree with a Prettier decision, the fix is to change
`.prettierrc.yaml` (and discuss the change with the user), not to fight
it file by file.

### Structure carries the story

Each file should make sense on a **first skim**. Headings and lists do
most of the work; long unbroken prose does not. When you find yourself
writing a third paragraph in a row with no heading or list between them,
ask whether the section actually has internal structure you should
expose.

Concretely:

- Use heading levels in order: `#`, then `##`, then `###`. Don't skip
  from `#` to `###`.
- Break related items into a list rather than burying them in prose.
- A table beats prose when you're describing parallel attributes of
  several things.

### Prefer clarity over decoration

- Use **bold** for the one or two things on a page that a skimmer truly
  needs to catch, not for every noun phrase. Bold loses meaning when
  it's everywhere.
- Use `inline code` for literal identifiers (file names, command
  fragments, code symbols), not for casual emphasis.
- Use blockquotes for quoted material, not as a styling effect.

### Links

Use **clickable Markdown links**: `[About](https://example.com/about/)`.
Do not paste bare URLs as backticked text or bold text and expect the
reader to copy-paste them. This applies to chat replies as much as to
docs.

When linking inside the repo, prefer relative paths
(`[README.md](README.md)`) so links survive moves and forks.

### Plain language

Avoid throwaway metaphors from unrelated domains — phrases like "land
it," "load-bearing," "in flight," "lift," "moving the needle," "north
star," "heavy lift," and other shallow aviation, construction, or sports
analogies. Use the direct verb: "commit it," "essential," "in progress,"
"effort," "changes the result," "goal," "hard."

## When checks fail

`./bin/check` failures fall into two categories. Handle each one
mechanically.

### Prettier check failed (`prettier --check` exits non-zero)

The file is not in canonical form. **Run `./bin/format` and re-run
`./bin/check`.** That's the entire response. Do not hand-edit the file
to try to match what you guess Prettier wants — you will lose, and you
will create churn in the diff.

If `./bin/format` is somehow making the file _worse_ (extremely rare,
usually a sign the file mixes Markdown with another format the formatter
doesn't understand), stop and report to the user rather than disabling
Prettier on the file.

### markdownlint failed

Read the error. Each line of output names a file, a line number, a rule
ID (e.g. `MD040`), and a short description. Fix what the rule reports.

Common fixes:

- **MD040** (fenced-code-language) — add a language tag to the fence
  (`sh`, `yaml`, `text`, etc.).
- **MD025** (multiple top-level headings) — demote all but one of the
  `#` headings to `##`.
- **MD024** (duplicate headings) — disambiguate the heading text.
- **MD001** (heading levels skip) — insert the missing intermediate
  level or restructure.

For the rare case where a rule genuinely should not apply to a specific
block (the git-commit skill's `text`-tagged example is one), prefer
scoped inline disables over global rule disables:

```markdown
<!-- markdownlint-disable MD040 -->

(content the rule should ignore)

<!-- markdownlint-enable MD040 -->
```

If you find yourself disabling the same rule in many files, that's a
signal to turn it off globally in `.markdownlint.yaml` and document why,
rather than peppering files with disables.

## Conversation startup (Markdown-heavy repos)

For small, tightly cross-referenced Markdown repos (planning docs,
design docs, internal wikis), **load every Markdown file in the
repository before doing substantive work.** Partial context produces
brittle answers and makes it easy to write a paragraph that duplicates
one already living in another file.

For larger repos where reading everything is impractical, at minimum
read `README.md`, `AGENTS.md` (or the equivalent), and any
index/table-of-contents file before editing anything.

## Quick reference

| Situation                  | Action                                                |
| -------------------------- | ----------------------------------------------------- |
| Just edited a `.md` file   | `./bin/format`                                        |
| About to commit            | `./bin/check` (must pass)                             |
| `prettier --check` failed  | `./bin/format`, then re-run `./bin/check`             |
| `markdownlint` failed      | Fix what it reports; never hand-edit to please it     |
| Tools not installed        | Detect environment, then Branch A / B / C above       |
| File should be exempt      | Add to `ignores:` in `.markdownlint-cli2.yaml`        |
| One block should be exempt | `<!-- markdownlint-disable RULE -->` / `-enable RULE` |
| Disagree with Prettier     | Change `.prettierrc.yaml`, not the file               |
| Want CI to enforce checks  | Add `.github/workflows/check.yml` (see CI section)    |
