---
name: incident-report
description: Investigate a failed or flaky test run, deploy or environment problem down to its mechanism, and write it up as a blame-free incident report that marks every claim as certain or inferred. Use when asked for "the real reason" a build failed, an incident write-up, a postmortem, a gist about what happened, or before accepting a timeout bump, retry or rerun as the fix.
---

# Incident report

The goal is one sentence that says what raced what, or what was missing when, backed by
timestamps from more than one source. The report is how that sentence reaches people who were not
in the investigation.

A report that ends in "probably flaky", "infra hiccup" or "slow CI" has not found the cause. Keep
going or say plainly that the cause is unknown and what would find it.

## Principles

- **A failure is a bug until proven otherwise.** Timeouts, retries and reruns hide it. The report
  exists so nobody has to rediscover it.
- **The cause is usually only visible in the correlation.** One source rarely shows it. The test
  output says *what* failed, the deploy log says *when* things changed, container logs and metrics
  say *what the system was doing*, config and code say *why it could happen*. Put them on one
  timeline.
- **Certain and inferred never mix.** Every claim is either read directly from a record, or it is
  a conclusion with its basis and a confidence level. A reader must be able to tell which is which
  without asking.
- **Raise certainty before you write.** When a claim is inferred and a record exists that would
  prove or refute it, go and read that record first. Write the report on what you found, not on
  what you guessed.
- **Try to break your own claims.** Look for a counterexample in the data before you publish a
  generalisation. "X always happens" needs every case checked, not the one you looked at.
- **Blame-free.** No personal names. Attribute to artifacts: PR numbers, build numbers, commits,
  config files. A PR that tried the wrong fix is "proposed in #1234", not a person.
- **Plain language.** Short sentences, simple words, no emojis. Simple but still accurate.

## Process

### 1. Pin the failure

- Get the exact failure text of every failed step, not just the first one. Strip color codes.
- Group failures by signature. Six failures with the same message in one two-minute window are
  one event, not six.
- Note each failure's time. Live markers during the run give the real time. The summary printed
  at the end of a spec only gives the time the spec finished.
- Work out when each failed wait started (failure time minus its timeout). The window that matters
  is when the waits began, not when they gave up.

### 2. Build the window

- Find what ran before the failure: the deploy or build that triggered the run, when it finished,
  and what the test harness waited on before starting.
- Convert every timestamp to UTC. Sources disagree: CI consoles, container logs and dashboards
  often use different zones. Say in the report which source used which zone.
- Pull the system side for the same window: which containers or tasks started, stopped or were
  replaced (a new task id in a log stream name means a new container), capacity and scaling
  metrics, queue and connection events.
- Read ephemeral records first. Some records, such as instance termination reasons, expire within
  an hour or a few weeks. Note what has already expired.

### 3. Diff against green

- Find the last green run and list every commit between it and the red one. Read each diff. Rule
  each commit in or out with a reason.
- Compare the same measurement across green and red runs: startup times, scaling steps, queue
  depths, step durations. The variable that differs on the red run is the lead.
- Check whether the check that failed is new. A new, stricter assertion can make an old problem
  visible for the first time. That is the assertion doing its job.

### 4. Test the known explanations

- List the explanations already on the table: previous incidents, the explanation in a proposed
  fix, "the CI machine was overloaded", "spot instances". Test each one against this incident's
  records and rule it in or out explicitly, with the record that decides it.
- A matching symptom is not a matching cause. Two incidents can fail the same assertion for
  different reasons.
- Look at where each piece actually runs in *this* environment. Config differs between
  environment types, and a fix in one may not apply to another.

### 5. Find the mechanism

- Read the config and code that decide the behaviour: sizing, placement, scaling settings, health
  checks, readiness gates, entrypoint scripts. Cite file and line.
- State the mechanism as a chain of facts, each with its source, ending at the failure.
- Separate the trigger (what was different today) from the gap (what let the trigger turn into a
  failure). The fix usually belongs on the gap.

### 6. Measure how often

- Go back as far as the records allow, not just the last few runs.
- Show the margin, not only pass/fail. A table of "how close it was" on each run shows a race that
  pass/fail hides.
- Watch for selection bias: events that happen when no test is running go unnoticed, so "it
  always happens during tests" is often just "we only notice it during tests".
- For a single event, say so. One occurrence supports a very wide range of rates.

### 7. Raise certainty, then self-check

Before writing, go through every inferred claim:

- Is there a record that would prove it? Read it.
- Is there a config file or code path that would explain it? Read it.
- Does any run in the data contradict it? Check them all.

Then list what is still inferred and why it can't be proven from here (record expired, source
not reachable, no logging for that event).

### 8. Decide the fix

- One primary recommendation, aimed at the gap, not the trigger.
- Show what it would have done in *this* incident, with timestamps, and in any related incident.
- Say where it lands (which repo or system) and roughly what it costs.
- List what does not fix it and why: longer timeouts, retries, reruns, workarounds that shift the
  race. Be concrete: "90s covers today, the gap is set by X, not by Y".
- Smaller follow-ups go in their own sections, marked as proposals if they are not designed yet.

## Output format

Markdown. Draft it as a file in the scratchpad first. Publish it only when asked, as a secret gist
unless told otherwise.

```
# <ENVIRONMENT> - <YYYY-MM-DD> incident

<Build/run> ended <RESULT> with <N> failed <scenarios/steps>. <One line on how they relate:
same error, same step, same window.>

All times UTC. **Certain** means read directly from a log line, a metric, the config or the code.
**Inferred** means a conclusion drawn from those, with what it rests on.

<Links to related reports, one line each on how this one differs or connects.>

## Summary

<The mechanism in a few short paragraphs. What was missing, since when, why nothing waited for it.>
<Where true: "The application, the tests and X all worked as designed.">

**Recommendation: <one line>.** See *Fix*.

## What happened on <date>

| time | event | source |
|---|---|---|
| hh:mm:ss | <quoted log text or metric value> | <source kind> |

<Quote the exact failure text once in a code block.>

**Certain**
- <fact, with where it was read>

**Inferred**
- <conclusion>. It rests on <records>. Confidence: <high/medium/low>.

## <Why it could happen: the mechanism section, named after the mechanism>

<Config and code facts with file names. Mark any inferred step.>

## <Why this run was different>

<Comparison table against green runs. Mark the inferred part.>

## How often

<Table over every run the records cover, with the margin. Note gaps and exclusions.>

## <What it has in common with related incidents> (when relevant)

## Fix

### 1. <Primary fix>
<What, where, a short snippet if it helps, what it would have done here with timestamps.>

### 2. <Follow-up> (optional, marked as proposal if not designed)

## What does not fix it

- **<Tempting fix>.** <Why not, concretely.>

## Open questions (when any remain)

<What is unknown, which record would answer it, and whether it can still be read.>

## Where to look next time

<The one comparison that would have found this fastest. Which log group or stream, which line to
compare against which.>

Not checked: <anything in scope that was not looked at>.
```

Rules for the format:

- Timeline rows are records. Anything derived goes in the Certain/Inferred lists or a
  "Derived" subsection, never in the timeline.
- Bold the two or three rows that carry the argument.
- Quote log text exactly, in backticks.
- Tables over prose for anything compared across runs.
- Never copy secrets, passwords, API keys or tokens from logs into the report, even when the
  source prints them. Mention the leak separately to the user instead.
- The footer can state how the report was built, e.g. which sources were correlated, when that
  was itself the finding.

## Before publishing

- Every Certain claim: can you point at the record? If not, move it to Inferred.
- Every Inferred claim: does it say what it rests on and how confident it is?
- Every "always", "never", "every time": checked against all runs in the data?
- Every number in a table: recomputed from the source, not from memory?
- No names, no secrets, no emojis.
- The fix section says what it would have done in this incident.
