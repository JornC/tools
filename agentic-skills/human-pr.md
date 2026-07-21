---
allowed-tools: Bash(git:*), Bash(gh:*), Bash(jira-api:*), Bash(jq:*), Read, Edit, Write, Grep, Glob, Skill
description: Human-styled AERIUS PR protocol - wrap up an in-session change and open its PR (the default for all PRs)
argument-hint: [AER-####] and/or your PR body text
---

## When to use

You've developed a change in this conversation - usually on a working branch, changes usually
already committed - and want to finish by opening the PR in the house style. This is the default for
any real PR: the work is whatever you built and reviewed in-session, so there is no size limit and no
minimal-change discipline. Use the branch you're already on - do not start a fresh one or reset -
then commit anything outstanding, push, and run the draft → create flow.

For a single tiny standalone fix made from scratch (a typo or rename, a one-or-two-line diff), use
`trivial-pr` instead - it adds the fresh-branch + minimal-change discipline this skill deliberately
omits. Same commit/PR conventions, same no-attribution rule, same draft-before-create gate, same
two-zone body either way.

## Inputs

`$ARGUMENTS` may be:
- An AERIUS issue key (e.g. `AER-4444`) - fetch context via the jira skill first
- Freeform description of the change - use it directly
- Your own PR body text - the words that go in the **top zone** of the body (see [The two-zone PR
  body](#the-two-zone-pr-body)), used near-verbatim and never rewritten. May also arrive from the
  conversation rather than the argument.
- Empty - work from the prior conversation context

If an issue key is present, invoke the jira skill (or run `jira-api` directly) to read the issue
before touching anything. The body usually names the exact strings or files to change.

## Remotes (assumed)

- `origin` = your fork (push target) - resolve the owner from `git remote`, don't assume a name
- `upstream` = AERIUS org (PR base, `upstream/main`) - the PR repo (`--repo aerius/<repo>`) is the
  `upstream` remote's owner/name

If those names aren't set this way, stop and ask.

## Roles

Two roles appear below, normally the same person:
- **the author** - whose words go in the top zone of the PR body
- **the user** - who drives this session and approves the draft before it's created

## The two-zone PR body

The body has up to two clearly separated zones, so a reader can always tell the author's words from
the AI's: the author's own text on top - short, direct, sometimes less precise - and anything
machine-written quarantined below the line in an openly labeled block. The aim is that AI prose is
never mistaken for his/her own - that is the thing to avoid.

**Top zone - the author's text (near-verbatim).**
The author supplies this, as the command argument or from his/her words in the conversation. Place
it almost exactly as given:
- Allowed edits: spelling, grammar, punctuation, and light style/consistency touchups. That is all.
- Not allowed: expanding, padding, restructuring, or rephrasing into different wording. Keep it
  short and direct - the terseness is the feature, even at the cost of some precision.
- Preserve his/her voice and his/her hedges ("seems like", "might have been"). Do not sand
  uncertainty off into confident claims.
- Plain text only in this zone: no headers, no bullets, no markdown ceremony.
- Fact-check it against the actual change before shipping (see step 4). If it contains a factual
  error, handle it per that step - never silently fold a correction into the body.
- A trivial change (typo, rename) may have an empty top zone - and then usually no bottom zone
  either.

**Bottom zone - the AI section (optional, openly AI).**
Below a `---`, inside a collapsed `<details>` block whose `<summary>` marks it as AI-written. GitHub
renders `<details>`/`<summary>` in PR bodies, so it stays collapsed until a reviewer expands it - the
`<summary>` line is the label.
- This is machine prose in the AI's own voice. It may be more technical, more verbose, more
  elaborate than the top. It may also be minimal when there is little to add - scale it to the
  change, never pad it.
- Write it in the AI's own voice. Do **not** write it in the author's voice or borrow his/her hedges
  - the whole point is that it reads as clearly not-him/her.
- Include it only when there is technical substance worth recording. It is never mandatory; trivial
  changes skip it, and skip it too if the author only wants his/her own text.

Shape:

```
<author's own text, lightly cleaned up>

---
<details>
<summary>LLM-generated technical summary</summary>

<technical prose in the AI's voice - as detailed as the change warrants,
or minimal when there is little to say>
</details>
```

Example - the author's terse line on top, the AI's fuller account collapsed below:

```
Fixes a flaky select-source test. Most likely just flake from the rollout animation;
nothing nearby changed.

---
<details>
<summary>LLM-generated technical summary</summary>

The "select source x" step clicks the source, which deselects it when it is already selected.
When that happens the following assertion can still pass during the ~150ms rollout animation,
while the close button remains clickable - the most likely explanation for the failure in run
660. The preceding runs all passed this test and nothing in the vicinity changed, so test flake
is the leading explanation over a real regression.
</details>
```

## Protocol

### 1. Branch

Stay on the feature branch you already developed on - do not create a new branch or reset. It should
be branched off `upstream/main`; if it's stale, that's the user's call, not a `reset`. If you're on a
branch with uncommitted work that belongs to the change, that's expected; if you somehow find
yourself on `main` with nothing to wrap up, stop and ask.

### 2. Commit

There are usually already commits from the session - just commit anything still outstanding (`git add
-A` is fine here); never `--amend` what's already there.

Style (AERIUS):
- Title: `AER-#### - {what}` - dash with spaces, lowercase `{what}`, terse noun phrase
- Body (optional, only if the *why* isn't obvious from the title): one or two short plain-text lines.
  No bullets unless genuinely listing distinct items. No headers. No "Changes:" preamble. No test
  plans.
- No Claude attribution - no `Co-Authored-By`, no "Generated with" footer (see Hard rules).
- No `--no-verify`, no `--amend`.

```bash
git add <specific-files>
git commit -m "AER-#### - {what}"
```

If a freeform task with no ticket: title is just `{what}` in the same terse style.

### 3. Push to origin

```bash
git push -u origin <branch-name>
```

### 4. Fact-check the top zone

Before drafting, read the author's top-zone text against what the change actually does - the diff,
the commit, the code. Two failure modes to catch - both would give a reviewer the wrong picture of
the change:

1. **Factually wrong** - a claim that is plainly incorrect: names the wrong file or mechanism,
   misstates the cause, says it does X when it does Y.
2. **Missing major content** - a distinct, significant part of the change that the text does not
   acknowledge at all, such that a reader comes away with the wrong idea of what the PR is.

What is NOT a failure, and must never be "fixed":
- Terseness, simplification, and honest hedges. The top zone is meant to be short and can trade
  precision for brevity. Never sand these off into verbose precision.
- Omitting detail. The AI section carries the fine-grained account; the top zone needs only a fair
  gist, not a changelog. Only *major* omissions count - a whole part of the change, not a nuance.

When you find either:
- Say so in chat, separately, with the specific problem and the smallest correction that fixes it -
  do not silently edit his/her text, and do not fold the fix into the body. He/she is the author;
  he/she decides whether to adjust the wording, keep it, or let the AI section carry the missing
  precision.
- If the top zone and the AI section contradict each other on a fact, that is a signal one of them
  is wrong - surface it rather than shipping both.

**On any mismatch between the author's text and the implementation, reconcile FIRST - never push
anything to force alignment.** A mismatch can mean the text is wrong, the implementation is wrong,
or both. Do not assume one side must yield to the other. Do not push *anything* to line them up -
not a rewrite of the text, and just as importantly not a change to the code / workflow / config to
match the words. Silently editing the implementation to fit the text is exactly as wrong as silently
editing the text to fit the implementation, and it may "fix" the wrong side (the text can be the
mistaken one). Stop, surface the mismatch in chat with both readings, and let him/her decide which
side is correct. Only after he/she decides do you touch either side, and only then push.

Always state the verdict explicitly - e.g. "read against the diff, the top zone is accurate and
complete enough" - so the check is visible and never silently skipped.

### 5. Draft the PR - do NOT create it yet

Show the user:
- Branch name
- Commit subject(s)
- Proposed PR title - same as the commit subject for a single-commit PR; for a multi-commit PR, a
  terse noun phrase covering the whole change in the same `AER-#### - {what}` style
- Proposed PR body - show both zones (see [The two-zone PR body](#the-two-zone-pr-body)): the
  author's near-verbatim text on top, then the collapsed AI section if there is one. The top zone is
  often empty or one line; never test-plan sections.
- The exact `gh pr create` command that would be run

Format the draft so the user can eyeball it in one screen. Example:

```
Branch: aer-4444-warmte-inhoud → upstream/main
Title:  AER-4444 - warmteinhoud naar warmte-inhoud
Body:   (empty)

Command:
  gh pr create --repo aerius/<repo> --base main --head <fork>:aer-4444-warmte-inhoud \
    --title "AER-4444 - warmteinhoud naar warmte-inhoud" --body ""
```

For a **non-empty** body - especially one with an AI `<details>` block - write the full body to a
temp file and pass `--body-file <file>`, not `--body`. Multi-line HTML does not survive inline
`--body` cleanly.

Then wait for the user's go-ahead.

### 6. Create the PR - only after explicit approval

Run the `gh pr create` command shown in step 5. Return the PR URL.

## Hard rules

- No Claude/tool attribution anywhere - no `Co-Authored-By`, no "Generated with Claude Code" footer,
  in commit messages, PR title, PR body, or branch names. This bans the *promotional* trailer, not
  the AI section of the body: that section is a deliberate content label, and its `<summary>` stays
  generic ("LLM-generated technical summary") - it never names or promotes the tool.
- The author's top-zone text is never rewritten or expanded - light copy-edits only (spelling,
  grammar, style). All AI prose stays below the line, in the labeled collapsed block, in the AI's own
  voice - never dressed up as the author.
- No emojis anywhere.
- No `--amend` and no force-push.
- Never `reset --hard` over a branch with work on it.
- Never run `gh pr create` until the user has approved the draft in step 5.
