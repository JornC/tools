---
name: incident-report
description: Investigate a failed or flaky test run, deploy or environment problem down to its mechanism, and write it up as a blame-free incident report that marks every claim as certain or inferred. Use when asked for "the real reason" a build failed, an incident write-up, a postmortem, or before accepting a timeout bump, retry or rerun as the fix. Human-driven: expect many rounds of questions, not a one-shot answer.
---

# Incident report

The goal is one sentence that says what raced what, or what was missing when, backed by
timestamps from more than one source. The report is how that sentence reaches people who were not
in the investigation.

A report that ends in "probably flaky", "infra hiccup" or "slow CI" has not found the cause. Keep
going, or say plainly that the cause is unknown and which record would find it.

Think of how an onion rots: from the center outward. To find the source you peel it layer by
layer. Halfway in you will find rot, and it is tempting to stop there and call it the cause. It
usually isn't. It is rot that spread from further in. A timeout that expired, a container that
wasn't there, a queue with no consumer: each is a layer. Keep peeling until you reach the layer
nothing else explains, and fix that one.

## Who drives this

This is not an autonomous skill. It will not solve an incident in one go. A human drives the
investigation, and the agent assists.

That comes from experience. Left alone, the agent writes a shallow report that stops at the first
or second layer and sounds finished. It is not good at doubting its own results. What gets to the
center is a developer who keeps pushing: who asks hard questions, doubts each answer, and works to
understand the processes involved themselves. Each layer usually takes another round of that.

So expect many rounds and many questions. For the agent, that means:

- Treat each answer as one layer, not the end. Say which layer you think you reached and what
  could still sit under it.
- Say what you did not check and which claims are weakest, so the developer knows where to push.
- When the developer pushes back, take it as the next step, not something to defend against. Go
  back to the records.
- Explain the processes involved plainly, so the developer can judge the mechanism themselves.
  They need to understand it, not only read the conclusion.
- Don't declare the investigation done. The developer decides when the center is reached.

## How to use this

This is a guide, not a script. Every incident is different. Pick the parts that fit, skip the
ones that don't, and change the order when the evidence points somewhere else. The developer
asking for the investigation can steer: "just the timeline", "only check the deploy", "skip the
frequency analysis", "we already know the trigger, find the gap". Follow that over anything here.

The same goes for the report. The format below lists sections that have been useful. A short
incident may need four of them. Leave out what adds nothing.

## Principles

- **A failure is a bug until proven otherwise.** Timeouts, retries and reruns hide it. The report
  exists so nobody has to rediscover it.
- **The cause is usually only visible in the correlation.** One source rarely shows it. The test
  output says *what* failed, the deploy log says *when* things changed, container logs and metrics
  say *what the system was doing*, config and code say *why it could happen*. Put them on one
  timeline.
- **Certain and inferred never mix.** Every claim is either read directly from a record, or it is
  a conclusion with its basis and a confidence. A reader must be able to tell which is which
  without asking.
- **Raise certainty before you write.** When a claim is inferred and a record exists that would
  prove or refute it, read that record first.
- **Measured beats argued.** A count of events in the logs over weeks beats a conclusion drawn
  from how the config ought to behave. When the two disagree, trust the count.
- **Try to break your own claims.** Look for a counterexample before you publish a
  generalisation. "X always happens" needs every case checked, not the one you looked at.
- **Explain the trigger, not only the gap.** A fix that holds whatever the trigger was is
  valuable, but it does not close the investigation. The trigger still needs a cause, or an
  explicit "unknown" plus the record that would answer it.
- **Blame-free.** No personal names. Attribute to artifacts: PR numbers, build numbers, commits,
  config files. A PR that proposed the wrong fix is "proposed in #1234", not a person. Don't shame
  anyone, including in examples of what not to do.
- **Plain language.** Short sentences, simple words. A reader who was not there should follow it.

## Ways in

Pick what the incident needs. Roughly in the order they usually pay off.

### Pin the failure

- Get the exact failure text of every failed step, not just the first one. Strip color codes.
- Group failures by signature. Six failures with the same message in one two-minute window are
  one event, not six. Failures with a different signature in the same run may be a second,
  unrelated incident. Say so and keep them apart.
- Find when each failure happened. Live progress lines give the real time; summaries printed at
  the end of a spec only give the time the spec finished.
- Work out when each failed wait started (failure time minus its budget). The window that matters
  is when the waits began, not when they gave up.
- Read the error text for *which* clock ran out. A wait can have more than one phase (e.g. waiting
  for a request vs. waiting for its response), and the message usually says which.
- Some budgets are implicit: a poll count times a delay, plus round trips. Measure the real budget
  from the run instead of reading it off the code.
- Don't trust a green summary. A report overview can drop a failed feature. Check the raw
  per-test results too.

### Check what was actually tested

- Which commit did the failing run build and test? A PR run may test a branch that is behind
  main, or a job may run main's tests instead of the PR's. Get the SHA from the run itself.
- Which version was deployed where? Claims like "it is redeployed every day, so it has the fix"
  need the deploy record, not an assumption.

### Build the window

- Find what ran before the failure: the deploy or build that triggered the run, when it finished,
  and what the test harness waited on before starting.
- Convert every timestamp to one zone and say which. Sources disagree: CI consoles, container logs
  and dashboards often use different zones.
- Pull the system side for the same window: which containers or tasks started, stopped or were
  replaced (a new task id in a log stream name means a new container), capacity and scaling
  metrics, queue depth and consumer counts, connection open and close events.
- Read ephemeral records first. Some records, such as task stop reasons or instance termination
  causes, expire within an hour or a few weeks. Note what has already expired.

### Diff against green

- Find the last green run and list every commit between it and the red one. Read each diff. Rule
  each one in or out with a reason.
- Compare the same measurement across green and red runs: startup times, scaling steps, queue
  depths, step durations. The variable that differs on the red run is the lead.
- Check whether the check that failed is new. A new, stricter assertion can make an old problem
  visible for the first time. That is the assertion doing its job.

### Test the known explanations

- List the explanations already on the table: earlier incidents, the reasoning behind a proposed
  fix, "the CI machine was overloaded", "the instance was reclaimed", the requester's own theory.
  Test each one against this incident's records and rule it in or out with the record that decides
  it.
- A matching symptom is not a matching cause. Two incidents can fail the same assertion for
  different reasons.
- Look at where each piece actually runs in *this* environment. Config differs between environment
  types, and a fix in one may not apply to another.
- Rule things out with numbers where you can: memory estimates against limits, request sizes, rates
  against published figures.

### Find the mechanism

- Read the config and code that decide the behaviour: sizing, placement, scaling, health checks,
  readiness gates, entrypoint scripts, retry and reconnect logic. Cite file names.
- State the mechanism as a chain of facts, each with its source, ending at the failure.
- Break a gap into measured phases where you can (placed, pulled, started, ready). "It took four
  minutes" hides which phase to fix.
- Beware of misleading metrics. Container memory for a JVM sized by a heap percentage looks full
  by design; CPU and heap after garbage collection say more.

### Reproduce it

The strongest evidence there is. When the mechanism can be reproduced locally, do it.

- Aim for the exact failure signature, not just "a failure".
- Find the threshold: how long a pause, how many readers, how tight a race before it fails.
- Show before/after in a small table: runs, outcomes by kind, with and without the fix.
- Prove any new or changed check can go red: break the thing it guards once and watch it fail
  when and how you expect.
- To find a race, make it fail more, not less: remove fixed waits, shorten timeouts.

### Measure how often

- Go back as far as the records allow, not just the last few runs.
- Show the margin, not only pass/fail. A table of how close each run came shows a race that
  pass/fail hides.
- Keep an inventory with explicit exclusions: which failures look similar but are something else,
  and why.
- Watch for selection bias: events that happen when no test is running go unnoticed, so "it
  always happens during tests" is often "we only notice it during tests".
- For a single event, say so. One occurrence supports a very wide range of rates.

### Raise certainty, then self-check

Before writing, go through every inferred claim:

- Is there a record that would prove it? Read it.
- Is there a config file or code path that would explain it? Read it.
- Does any run in the data contradict it? Check them all.
- Would someone challenge it? Add the link or the exact source.

Then list what is still inferred and why it can't be proven from here: the record expired, the
source is not reachable, nothing logs that event. When a record held by someone else would settle
it, give them the exact query and the string in the result that decides the question. Asking
others for records costs them time, so do it when you have a strong lead.

### Decide the fix

- The Fix section holds what would have prevented *this* incident. Show what it would have done
  here, with timestamps, and in any related incident.
- Prefer fixes that make the system predictable over fixes that tolerate unpredictability. Waiting
  for a known-good state beats waiting longer.
- Say where it lands (which repo or system) and roughly what it costs.
- Don't propose capacity or infrastructure changes before the mechanism is confirmed.
- List what does not fix it and why: longer timeouts, retries, reruns, workarounds that move the
  race. Be concrete: "90s covers today; the gap is set by X, not by Y".
- If the trigger could not be found, add what would make the next one answerable: capture the
  records that expired this time, add the log line that was missing.

## Certainty in the report

- **Certain:** read directly from a log line, a metric, a report, a reproduction, the config or the
  code. Say where.
- **Inferred:** a conclusion. Say what it rests on, and give a confidence in words (high, medium,
  low). Never give a percentage unless it was measured.
- **Best guess:** when the cause stays unknown, give one best guess, clearly labelled, and what
  would confirm or refute it. Don't weight several guesses against each other without data; that
  is guessing dressed as analysis.
- **Open question:** anything unexplained that doesn't fit, even if it doesn't change the fix. Keep
  it named instead of smoothing it away.
- Word links to earlier reports carefully. "Would also explain" is honest; "fully resolves" needs
  proof.

## Report format

Markdown. Draft it as a local file first, named the way it should appear once published. Publish
only when the developer asks, where they ask. Publishing makes it visible to others.

Pick from these sections. Keep the order; drop what adds nothing.

```
# <ENVIRONMENT> - <YYYY-MM-DD> incident

<Build/run> ended <RESULT> with <N> failed <scenarios/steps>. <One line on how they relate.>

All times UTC. **Certain** means read directly from a log line, a metric, the config or the code.
**Inferred** means a conclusion drawn from those, with what it rests on.

<Links to related reports, one line each on how this one differs or connects.>

## TL;DR

- Cause: <one plain sentence>
- Fix: <one plain sentence>
- Root-caused: <yes / trigger unknown, gap known / no>
- Impact outside test environments: <none / what users would see>

## Summary

<The mechanism in a few short paragraphs.>
<Where true: "The application, the tests and X all worked as designed.">

## What happened on <date>

| time | event | source |
|---|---|---|
| hh:mm:ss | <quoted log text or metric value> | <kind of source> |

<The exact failure text once, in a code block.>

**Certain**
- <fact, with where it was read>

**Inferred**
- <conclusion>. It rests on <records>. Confidence: <high/medium/low>.

## <The mechanism, named after it>
## <Why this run was different>        (comparison table against green runs)
## Reproduction                        (when there is one)
## How often                           (every run the records cover, with the margin)
## Considered and ruled out            (each with the record or number that rules it out)
## Fix
## What does not fix it
## Open questions / best guess         (when the trigger is unknown)
## Where to look next time             (the one comparison that finds this fastest)

Not checked: <anything in scope that was not looked at>.
```

Rules for the format:

- Timeline rows are records. Anything derived goes in the Certain/Inferred lists or a "Derived"
  subsection, never in the timeline.
- Bold the two or three rows that carry the argument.
- Quote log text exactly, in backticks.
- Tables over prose for anything compared across runs.
- Keep it as short as the incident allows. Trim before publishing.
- Never copy secrets, passwords, API keys or tokens from logs into the report, even when the
  source prints them. Report the leak separately.
- A closing line on how the report was built (which sources were correlated) is worth adding when
  that correlation was itself the finding.

## Before publishing

- Every Certain claim: can you point at the record? If not, move it to Inferred.
- Every Inferred claim: does it say what it rests on and how confident it is, in words?
- Every "always", "never", "every time": checked against all runs in the data?
- Every number in a table: recomputed from the source, not from memory?
- Does every item under Fix prevent this incident?
- Is the impact outside test environments stated?
- No names, no secrets, nothing that shames anyone.
