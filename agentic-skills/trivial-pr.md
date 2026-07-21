---
allowed-tools: Bash(git:*), Bash(gh:*), Bash(jira-api:*), Bash(jq:*), Read, Edit, Write, Grep, Glob, Skill
description: AERIUS PR protocol for one small standalone change - a typo/label/mechanical fix made from a fresh branch, kept minimal
argument-hint: [AER-####] and/or the change to make
---

## When to use

A single small standalone change - typo fixes, label renames, single-paragraph doc updates,
mechanical search-and-replace. The kind of thing where a reviewer sees one or two lines of diff and
immediately knows it's right. The change is made *under* this protocol, from a fresh branch, kept
minimal.

If it grows beyond a handful of obvious lines - new behavior, restructuring, anything where the right
answer isn't obvious - STOP and hand back. This skill is not for that: develop it properly and finish
with `human-pr` instead.

Same house style as `human-pr`: same commit/PR conventions, same no-attribution rule, same
draft-before-create gate, same two-zone body. The difference is only the front end - this skill
starts a fresh branch and holds the change to a minimal diff.

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
- Fact-check it against the actual change before shipping (see step 5). If it contains a factual
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
- Include it only when there is technical substance worth recording. It is never mandatory. Because
  this skill is for trivial changes by definition, the AI section is usually absent - or, when a
  small change still hides a detail worth a note, a single line such as "nothing special to add" or a
  one-liner on that detail. Scale it to the triviality; skip it too if the author only wants his/her
  own text.

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

Fresh branch off upstream/main.

```bash
git fetch upstream
git checkout -b <branch-name> upstream/main
```

Branch name: short kebab-case derived from the task. For an AER ticket, lowercase the key as a prefix
(e.g. `aer-4444-warmte-inhoud`). If currently on a branch with uncommitted work, STOP - don't reset
over it.

If a branch with this name already exists locally, pick a different name or ask. Never `reset --hard`
over an existing branch without confirmation.

### 2. Make the minimal change

Rules - these are the whole point of this skill:
- Only touch what the task literally asks for. No drive-by fixes, no neighboring cleanup.
- No new files unless the task requires it. No README or doc additions.
- No comments explaining the change. The diff and commit message are enough. If a comment genuinely
  must exist (e.g. a one-line doc on a new shared constant), keep it short - one line, no backstory,
  no ticket references.
- No reformatting of untouched lines.
- No "while I'm here" refactors.

If the task is "fix this typo in these three places", the diff should be three lines. If it grows
beyond that, STOP and tell the user - the task probably isn't a fit for this skill; develop it
properly and use `human-pr`.

### 3. Commit

A single focused commit for the change you just made.

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

### 4. Push to origin

```bash
git push -u origin <branch-name>
```

### 5. Fact-check the top zone

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

### 6. Draft the PR - do NOT create it yet

Show the user:
- Branch name
- Commit subject(s)
- Proposed PR title - same as the commit subject (a trivial change is a single focused commit)
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

### 7. Create the PR - only after explicit approval

Run the `gh pr create` command shown in step 6. Return the PR URL.

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
- Never run `gh pr create` until the user has approved the draft in step 6.
- If the change grows beyond "small and uncontroversial," stop and hand back to the user - develop it
  properly and use `human-pr` instead.
