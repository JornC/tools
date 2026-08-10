---
allowed-tools: Bash(git diff:*), Bash(git log:*), Bash(git branch:*), Bash(git merge-base:*), Bash(git rev-parse:*), Bash(gh pr view:*), Bash(gh pr diff:*), Read, Edit, Grep, Glob, Agent
description: Trim comments in the current changeset toward self-documenting code; relocate real rationale to a suggested commit message
---

## Context

Branch: !`git branch --show-current`
PR status: !`gh pr view --json title,body,baseRefName,url 2>&1 || echo "No PR found for this branch"`

## Philosophy

The goal is self-documenting code: code simple enough that it doesn't need a comment to be
understood. A comment earns its place only when the code genuinely cannot be made clear enough on
its own.

There are two different kinds of "why," and they go in two different places:

- **Why the code is shaped this way, permanently** - a hidden constraint, a subtle invariant, a
  workaround for a specific external bug, something that would surprise a future reader. This is the
  only thing allowed to live in a comment, and only when the code itself can't express it.
- **Why the change was made** - alternatives considered, what was tried first, the reasoning behind
  picking this approach, deliberation. This is history, not documentation. It belongs in the commit
  message. `git blame` should be what answers "why did this change," not a comment sitting next to
  the code.

Never leave in place:
- Comments that restate what the code already says.
- Comments that narrate the author's thought process ("decided to...", "tried X but switched to Y
  because...", "not sure if this is right but...").
- Comments referencing other, unrelated projects, repos, or code - they rot the moment that context
  changes and mean nothing to a reader who wasn't there.
- Commented-out code, changelog-style comments, apologies, hedges, section-banner noise.

Always defer to a project's own documented comment conventions (its CLAUDE.md / AGENTS.md) when
they set a different or stricter bar than this default - e.g. required Javadoc on public APIs,
license headers, mandatory linter-directive justifications, generated-code markers. Those survive
regardless of this rubric.

## Task

### 1. Determine the changeset scope

- If a PR exists for this branch, use its base ref (`baseRefName` above).
- Otherwise fall back through common bases via `git merge-base`: try `upstream/main`,
  `origin/main`, `main`, `master`, whichever exists.
- Full diff: `git diff <merge-base>...HEAD`

Scope is comments **added or modified by this diff** (they appear on `+` lines). Do not touch
comments elsewhere in a touched file that this diff didn't add or change - editing those is
unrelated-change scope creep. If one is egregiously bad, note it, don't fix it.

### 2. Extract every comment touched by the diff

Across whatever comment syntax the changed files use (`//`, `#`, `/* */`, `--`, `"""`, `<!-- -->`,
etc). For a large diff (many files), fan out one `Agent` call per file or small group in parallel;
for a small diff just do it inline. Every judgment needs the actual surrounding code in view - never
classify a comment from the diff hunk alone if the file has more context worth checking.

### 3. Classify each comment into exactly one bucket

- **Keep** - explains a genuinely non-obvious, permanent why. The code can't say this on its own.
- **Shorten** - same as Keep, but padded. Compress to the one clause that carries the actual
  information; cut the rest.
- **Cut + relocate** - the comment is deliberation or decision history: interesting, but it's about
  the *change*, not the code. Delete it from the file, carry its substance into the commit-message
  draft in step 5.
- **Cut + discard** - adds nothing anywhere. Restates the code, states the obvious, references
  unrelated code/projects, is commented-out code, is boilerplate/apology/banner noise. Delete, keep
  nothing.

### 4. Apply edits

Edit the comments directly per the classification above. Touch only comments - never change code
logic, formatting, or anything outside the diff's own scope. Keep each edit minimal: a Shorten edit
changes only the comment text, not its surrounding code.

### 5. Draft commit-message notes from everything Cut + relocated

Turn the relocated rationale into short bullets, phrased for a commit body - plain text, no
headers, matching how this repo actually writes commit messages (check `git log` for the local
style). This is a draft for the user to use, not something to commit.

### 6. Do not commit, amend, or push

This command edits working-tree files only. Never run `git commit`, `git commit --amend`, or any
push - those are for the user to do, and amending an existing commit violates linear-history rules
regardless.

## Output

For each file touched, a compact before/after per comment with its bucket and one-line reason.
Then, if anything was Cut + relocated, a clearly separated block:

```
## Suggested commit message notes
(paste into your commit message if you want this history kept - not applied automatically)

- <bullet>
- <bullet>
```

If nothing was relocated, omit that block entirely rather than printing it empty.
