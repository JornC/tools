---
allowed-tools: Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh api:*), Bash(git log:*), Bash(git diff:*), Bash(git branch:*), Bash(git merge-base:*), Bash(git rev-parse:*), Read, Write, Grep, Glob, Agent
description: Review current branch changes using expert code review principles
---

## Context

Branch: !`git branch --show-current`
PR status: !`gh pr view --json title,body,baseRefName,url 2>&1 || echo "No PR found for this branch"`

## Review Principles

See [review-principles.md](review-principles.md) for the full rationale behind each lens below. The
lenses are also self-contained - each carries its own mandate and calibration - so the review runs
without it.

## Task

You are orchestrating a multi-lens code review. Each lens is reviewed by a dedicated sub-agent that hyperfocuses on that lens only - a single agent trying to hold all lenses in scope invariably skims the ones that don't match its current thread of thought. Your job is scope setup, parallel dispatch, and synthesis. **Do not perform the lens reviews yourself.**

### 1. Collect scope artifacts ONCE (don't make sub-agents re-do this)

Run all of these and capture the output:

- Merge base: `git merge-base upstream/main HEAD`
- If a PR exists, read its title, body, and base branch (use the PR's base if it differs from `upstream/main`).
- Commit log: `git log --oneline <merge-base>..HEAD`
- Diff stat: `git diff --stat <merge-base>...HEAD`
- **Full diff**: `git diff <merge-base>...HEAD > /tmp/review-<sanitized-branch>.diff` - write it to a file once; sub-agents Read the file rather than re-running `git diff`.

This collected block is the **shared scope context**. Every sub-agent gets the same block and the same diff-file path.

### 2. Spawn one sub-agent per lens, in parallel

Send a single message containing one `Agent` tool call per lens (`subagent_type: general-purpose`). Each sub-agent's prompt must include:

1. The shared scope context (branch, merge base, PR title/body, commit log, diff stat).
2. The path to the pre-collected diff file. Instruct them to Read it first, and only `Read`/`Grep`/`Glob` the working tree for surrounding-system context (existing patterns, related files, conventions). **They should NOT re-run `git diff`.**
3. **Only their lens's mandate** (below). Do not leak other lenses' rubrics into their prompt.
4. The findings format from §3.
5. Explicit stay-in-lens instruction. If they notice something outside their lens, they may note it briefly under "Out of lens" but must not expand.
6. **Prohibited shell commands - pass this verbatim to every sub-agent:** The reason this rule exists is `.gitignore` awareness - plain `grep`/`find`/`ls`/`rg` recurse into `node_modules` / build artifacts / vendored deps and blow up context, and trigger permission prompts that break flow. Permitted (use these): the dedicated **Grep** and **Glob** tools (preferred - they respect `.gitignore`), and as functionally-equivalent fallbacks `git grep` and `git ls-files` (covered by `Bash(git:*)` allow). For file content, use **Read** (not `cat`/`head`/`tail`). Forbidden as Bash commands: standalone `grep`, `rg`, `find`, `ls`, `cat`, `head`, `tail`, `file`. Don't pipe `git grep` output through `head`/`tail`/`grep` either - use its own flags (`-n`, `-m`, `--max-count`) or accept the full output. This applies to every search, listing, and file-read operation the sub-agent performs.
7. **Universal posture (pass this to every sub-agent verbatim):** Returning "no findings" is a valid outcome - many diffs won't trip your lens, and saying so plainly is the right answer when nothing's wrong. Do NOT pad your output with stylistic preferences, alternative phrasings, or speculative concerns just to justify having looked. The bar for surfacing a substantive finding is "I'd want the author to see this." Anything below that bar - pedantic preferences, things that don't actually matter, "I'd have spelled this differently" - goes in the **Picky** bucket (see §3), which the synthesizer drops by default. Contribute Picky items only because you noticed them and they may be optionally available; don't contribute them because you think the user should act on them.

**Lenses:**

#### Lens 1 - Approach & Feature Fit
Challenge the premise. Does this change need to exist? Is it building something the system already provides in a different form? Is it working *with* the existing design or bolting on around it? Is it more complex than the problem warrants? Does the approach survive foreseeable evolution (new countries, re-uploads, replayed queue messages, added fields)? A correct implementation of the wrong approach is still wrong - say so even when the code is clean.

#### Lens 2 - Architecture & Layering
Layer violations, boundaries, placement. Does logic live in the right layer? Do domain objects carry framework/serialization concerns they shouldn't? Does UI-lifecycle-bound state (polling, tracking, async) survive navigation, or is it stranded in a component? Are internal IDs (auto-increments, DB PKs) leaking through APIs instead of stable external identifiers? Package cycles? Types that should move to a shared location because they cross layers?

#### Lens 3 - API & Schema Design
APIs, schemas, migrations, constraints, indices. Do responses expose more than consumers need? Are overloaded endpoints masking semantically separate operations? Do OpenAPI descriptions add signal or restate field names? Are migrations append-only (never edited)? Do constraints and indices justify themselves against actual queries? Do composite PKs permit the intended cardinality? Do triggers operate on the correct scope? For frontend-only PRs, apply this lens to **component prop surfaces** - overloaded props, mode flags, prop-vs-slot decisions.

#### Lens 4 - Naming & Semantics

**High bar.** Only flag a name when it is *actively misleading* - it tells a reader something untrue about what the thing is, or it will mislead a future maintainer once the context shifts. Stylistic preferences, "could be slightly clearer", alternative phrasings, and bikeshedding go in **Picky**, not in substantive findings. If a finding amounts to "I'd have called it something else", it's Picky at most.

Concrete things that *do* qualify:

- **Type/semantic mismatch.** A name claims one thing, the value is another: `createdDate` that is actually a timestamp; `reportFile` that is the record's main file, not a report; `clientSpecifics` on a field not specific to clients.
- **Boolean polarity inverted.** `true` should be the expected/default behavior. `disableCache: false` is wrong shape; rename to `enableCache: true`.
- **Name claims a guarantee the impl doesn't keep.** `findActiveId` that returns "the first pending record" - caller will assume the wrong contract.
- **Established convention violated** in a way readers will trip on (codebase uses `IT` suffix for integration tests and this one doesn't; codebase uses `useXyzStore` for stores and this is `xyzStore`).

Things to skip even if they bug you: a slightly long name; a name you'd have shortened; a name that's accurate but plain; "Collector" vs "Resolver" debates when both are defensible; lambda variable names in short scopes.

**Self-qualification gate - do this FIRST.** Skim the diff for names. For each candidate, ask: "would a reader form a false belief about this thing because of this name?" If no, skip it. If you cannot point to a specific false belief the name induces, return "no findings" with a one-line note. **Naming concerns, when flagged, default to Quick fix** - they only escalate to Question or Needs-attention when the misleading name is causing actual downstream confusion (wrong consumer assumptions, contract drift, test miscoverage).

#### Lens 5 - Correctness, Testing & Duplication
Real-input behavior: edge cases, error handling, null/undefined, concurrency, replay, idempotency, boundary conditions. Tests: specific assertions (not "something warned"), static/debuggable test data (not random UUIDs), assertion messages, `IT` suffix for integration tests, no mock-only coverage masquerading as real. Duplication of knowledge: values or state stored separately that could be derived from existing sources of truth.

#### Lens 6 - Hygiene
Formatting, unused code, visibility, consistency with codebase conventions, documentation necessity. Missing trailing newlines, redundant consecutive blank lines, trailing comments that should be above the line. Unused imports / variables / methods. `public` that could be package-private or private. Comments that restate what code already says. Generated-looking or AI-flavored documentation. Inconsistent import style (star imports, fully-qualified names, `var` vs explicit types, `useXyzStore` naming drift). Unrelated changes polluting the diff.

#### Lens 7 - Security & Trust Boundaries

Vulnerabilities at trust boundaries: untrusted input crossing into rendering / persistence / execution sinks, auth/authz changes, secrets and credential handling, network egress with user-controlled targets, deserialization. Think in categories - where is the boundary, what's the threat model for this code path, would a hostile input reach a sink?

**Self-qualification gate - do this FIRST.** Skim the diff for security-relevant signals in each layer touched (frontend: rendering of user strings, dynamic redirects, token storage; backend: SQL/HQL string building, security filters, file/URL handling, deserialization; infra/migrations: secrets, IAM, sensitive columns). **If no signals appear in the relevant layer, return "no findings" with a one-line note saying which categories you checked.** Do not fabricate concerns to justify the lens - most diffs have nothing for this lens, and that's the right answer.

When signals do appear, dig in: trace the untrusted input from source to sink; check that the boundary enforces the constraint the threat model requires.

### 3. Findings format (each sub-agent returns this)

```
## Lens: <name>

### Blockers
- `path/to/file:line` - one-line rationale
(or "None.")

### Questions
- `path/to/file:line` - ask-don't-tell phrasing ("Wouldn't it make more sense to ...?")
(or "None.")

### Good
- `path/to/file` - acknowledgment
(or "None.")

### Picky (synthesizer drops by default - surface only if everything else is empty)
- `path/to/file:line` - pedantic note in ≤10 words (preference, alt phrasing, "I don't actually care")
(or "None.")

### Out of lens (brief notes only)
- ...
(or "None.")

### No findings
If the lens had nothing substantive to flag, say so.
```

### 4. Synthesize and assign severity

After all sub-agents return, group findings by topic - the same concern flagged by multiple lenses becomes one item. Assign each item one severity:

- **Critical** - must fix before merge. Real bugs, correctness defects with a traced crash/breakage path, security issues with confirmed exploitability, regressions of existing behavior.
- **High** - should fix before merge unless explicitly deferred. Latent invariants with no guardrails, serious architectural smells that will be expensive to reverse, missing tests on load-bearing new logic.
- **Medium** - worth deciding before merge. Real tradeoffs where the reviewer can't pick without your context (consumer expectations, product intent, parked-for-next-PR judgment). The item has an actionable choice attached, not just "is this OK?".
- **Low** - polish. Dead code, naming, hygiene, minor cleanups. Always one-liners.

**Map from sub-agent buckets:** sub-agent "Blockers" usually become Critical or High. Sub-agent "Questions" usually become Medium (or Low if trivial). Sub-agent "Picky" gets dropped by default (see below).

**Hedging → Medium, not High.** If your synthesis prose is naturally hedging ("I'm not sure if...", "this might be wrong because..."), the item is Medium with a `Decide:` line, not High with an `Action:` line. Forcing a hedged thought into a declarative finding produces false confidence and wastes review attention. If you can't articulate the doubt at all, drop the item.

**Picky-bucket handling.** Sub-agents return a "Picky" bucket - pedantic preferences they noticed but explicitly don't want acted on. **By default, drop the entire Picky bucket.** Only exception: if the report would otherwise be entirely empty (zero Critical/High/Medium/Low), render `## Picky (you can ignore)` with one-line items and note in Overall that nothing substantive surfaced.

**Filter pedantry yourself - the synthesizer is a gate, not a passthrough.** Before rendering an item, ask: does this need the user's eye, or is this something I (the synthesizer) can handle, dismiss, or absorb? Examples of things to filter rather than surface:
- A sub-agent flagging a comment that "could be shorter" - drop, it's noise.
- A sub-agent suggesting a method "might be cleaner if split" - drop unless there's a concrete reason.
- Multiple lenses raising the same trivial naming concern - drop entirely, don't even render once.
- A sub-agent asking "wouldn't it be better to X" where X is a defensible alternative but the current code is also defensible - drop.
- A sub-agent surfacing an architectural concern the PR body already explicitly parks for a next PR - drop unless the parking is wrong.
- A sub-agent flagging missing tests where the codebase pattern is light testing in this layer - drop unless the new code is load-bearing.

The bar for surfacing anything is "I'd want the user to see this." If it doesn't clear the bar, the synthesizer eats it silently. The user does not want a comprehensive enumeration of everything noticed; he/she wants the actionable subset, severity-ordered. A clean report with 3 substantive items is better than a thorough report with 12 where 9 are noise.

**Calibration:** if an item collapses to "rename this," it's Low - title + file:line + ≤10 words. If a Medium item resolves into "do X," promote to Critical/High (it's a position, not a question). Err toward brevity; short, honest report is the goal.

### 5. Output structure

Single readable pass, severity-categorized. No rounds, no pagination, no separate "Questions" section. Every Critical/High/Medium item has the same shape - title, location, brief diagnosis, an Action or Decide line that tells the reader what to do.

```
Found <N> items - <C> critical, <H> high, <M> medium, <L> low.

## Critical
### <Title> - `path/to/file:line`
<1-3 sentences for a cold reader: what the code does, what's wrong, what breaks. Name symbols the first time they appear.>
**Action:** <Concrete fix sketch, ≤2 sentences. Not abstract - "Add null check and route to onFailure," not "handle the null case.">

(repeat per item, or "None.")

## High
### <Title> - `path/to/file:line`
<1-3 sentences.>
**Action:** <Fix sketch, or "Options:" with 2-3 paths if more than one defensible fix exists.>

(repeat, or "None.")

## Medium
### <Title> - `path/to/file:line`
<1-3 sentences: the context and the choice. State the tradeoff plainly.>
**Decide:** <What to weigh. Name 2-3 options and the trade-off, picking one if you have a defensible preference. Make explicit that the call is the user's.>

(repeat, or "None.")

## Low
- `[hygiene]` `path/to/file:line` - <≤10 words: what to change>
- `[naming]` `path/to/file:line` - <≤10 words>
- `[nit]` `path/to/file:line` - <≤10 words>
...

(or "None.")

## Overall
<One short paragraph. What this PR does and your candid read - clean and tight, working with the existing design or fighting it, right approach or wrong one. If it's a good change, say so plainly; if it's a slog, say that. Call out genuinely good patterns worth keeping if any stand out.>

## Lens coverage
| Lens | C | H | M | L |
|---|---|---|---|---|
| Approach & Feature Fit | 0 | 1 | 2 | 0 |
| Architecture & Layering | ... | ... | ... | ... |
...
```

**Title:** noun phrase or short imperative - "OrderSummary returned partial", "Drop unused syncLegacyData", "Multi-flag silent overwrite". Not a full sentence, not a question.

**Location:** `file:line` or `file:line-range`. Forward-slash paths. If an item spans multiple sites, list the primary in the header and mention the others in the diagnosis.

**Diagnosis (1-3 sentences):** written for a cold reader. Open with what the code is so the reader doesn't need to flip to the file. Name symbols the first time they appear; don't say "the new method" without naming it. State what's wrong directly - no "I think," "maybe," "could be" hedging. If you're hedging, the item is Medium not High.

**Action line (Critical/High):** concrete fix. Not abstract. Bad: "handle the null case." Good: "Add `if (payload == null) { onFailure(...); return; }` at the top of the success handler." If there are multiple defensible fixes, lead with `Options:` and name 2-3 with the trade-off, then pick one.

**Decide line (Medium):** name the choice + 2-3 options + trade-offs. The synthesizer can have a preference; the reader makes the call. Bad: "Is this the right approach?" Good: "Either (a) fold into the shared restore util now and parameterize on data source, or (b) keep two paths with a comment marking the duplication as transitional. (a) reduces future drift; (b) is cheaper for this PR. (a) if the next PR will need the shared util anyway."

**Low items (one-liner each):** `[tag]` `path/to/file:line` - ≤10 words. Tags: `[blocker]`, `[bug]`, `[minor]`, `[hygiene]`, `[naming]`, `[nit]`, `[doc]`, `[test]`. No paragraph rationale - trust the reader to greenlight.

### 6. Do NOT post comments to GitHub

Present the synthesized review to the user. He/she decides what gets submitted.
