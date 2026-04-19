# Proof CLI Active DX Concerns

Date: 2026-04-14
Author: Codex
Scope: Only concerns that still feel confusing after the latest validation pass. Fixed issues are intentionally excluded.

## Goal

Capture the remaining UX/DX friction from dogfooding `proof` on a real non-`proof` project, with:
- concrete examples
- why the behavior is confusing in practice
- concrete proposals, not just complaints

## Summary

The current CLI is now strong enough for real work. I was able to:
- understand the project using only the CLI
- drive requirement/test/code traceability to a clean state
- find real implementation bugs
- fix those bugs
- end at `proof audit --scope full` with `Errors: 0, Warnings: 0`

The remaining issues are not about missing fundamental capability. They are about operational clarity, statefulness, and helping users choose the right next step faster.

## 1. Shared `.proof/` State Is Easy To Misuse

### What happens

Several commands produce and consume the same artifact set under `.proof/`, especially:
- `.proof/coverage/unit.coverprofile`
- `.proof/test-results/go-test.json`
- `.proof/mcdc/...`

From a user perspective, many commands look stateless. In reality, they are coordinated through a shared mutable on-disk workspace.

### Why this is confusing

A user may reasonably assume these commands are independent:
- `proof audit --scope full`
- `proof workflow check --stage implement`
- `proof mcdc report --view hotspots`

But they are not independent if they read and write the same artifact paths.

### Concrete examples

#### Example A: concurrent audit and workflow check

```bash
proof audit --scope full
proof workflow check --stage implement --verbose
```

If these run concurrently in the same repository and both refresh `.proof` artifacts, one command can observe files while the other is still truncating or rewriting them.

Observed failure modes include messages like:
- `coverage profile contains no coverage blocks`
- `no test results found`

That looks like a product or project failure, but it may only be an artifact race.

#### Example B: running a report command before a fresh producer command

```bash
proof mcdc report --view hotspots
```

This command reads persisted MC/DC data from `.proof/mcdc/...`. If no measurement exists yet, or another command just removed or is regenerating that directory, the report fails.

That behavior is valid, but easy to misread as “MC/DC is broken” instead of “the report has no current backing artifact.”

#### Example C: manual cleanup during another proof run

```bash
rm -rf .proof/coverage .proof/test-results .proof/mcdc
proof audit --scope full
```

If another proof command is running at the same time, this can create false parse failures or partial-file reads.

### Why it matters

This kind of failure is expensive because it looks like a correctness problem in either:
- the project
- the test suite
- the CLI itself

but the true issue is just artifact concurrency.

### Concrete proposals

#### Proposal 1: explicit artifact-locking

If a command is regenerating shared artifacts, take a lock and either:
- block until the lock is released, or
- fail fast with a clear message

Example message:

```text
proof is currently regenerating shared artifacts in .proof/.
Running another artifact-producing command concurrently in the same workspace is not supported.
Active producer: proof audit --scope full
```

#### Proposal 2: staged temp directories with atomic promotion

Instead of writing directly into final artifact paths, write into per-run temp directories and only promote on success.

Example:
- `.proof/runs/<run-id>/coverage/...`
- `.proof/runs/<run-id>/test-results/...`
- `.proof/runs/<run-id>/mcdc/...`

Then update the canonical `latest` pointer or copy into final paths atomically.

This reduces partial-file visibility and race confusion.

#### Proposal 3: command help should state shared-state behavior explicitly

Relevant commands should say this plainly:

```text
This command regenerates shared artifacts under .proof/.
Do not run it concurrently with other proof commands that also refresh coverage, test-results, or mcdc artifacts in the same workspace.
```

#### Proposal 4: add an artifact status command

Example:

```bash
proof artifacts status
```

Desired output:
- which shared artifacts exist
- when each was produced
- which command/stage produced it
- whether it is currently locked/in-progress
- whether it is stale relative to the current working tree

That gives users a concrete model instead of forcing inference.

#### Proposal 5: make report commands self-diagnosing

When a report command depends on persisted state, the failure should say whether the problem is:
- no artifact exists yet
- artifact exists but is incomplete
- artifact exists but is stale
- artifact is currently being regenerated

That is much more actionable than a plain file-open or parse failure.

## 2. `suspect_clean` / `trace review` Still Feels More Stateful Than It Should

### What happens

The trace model is semantically defensible: links become suspect when linked artifacts change and previous reviews are no longer valid for the new fingerprint pair.

The difficulty is that this is operationally subtle.

### Why this is confusing

When a user sees suspect links after editing code, several different realities are possible:
- the link is genuinely wrong now
- the link is still correct, but review state is stale
- the link is still correct, but a different artifact changed and invalidated the review
- the user just needs to refresh review state

All of those can produce a similar first impression: “traceability is broken again.”

### Concrete example

A typical user experience is:

```bash
proof audit --scope full
```

They see:
- `suspect_clean` warning

Then they run:

```bash
proof trace suspect
```

And now they have to decide whether to:
- edit annotations
- run `proof trace autolink`
- run `proof trace review --all`
- ignore it because the link is still semantically correct

That decision is not obvious enough from the warning alone.

### Why it matters

This is one of the main places where the CLI still feels like it expects internal model knowledge from the user.

### Concrete proposals

#### Proposal 1: classify suspect links by repair type

Instead of only showing “review invalidated by linked artifact change,” classify each suspect link into a more actionable category such as:
- `review_refresh_likely`
- `annotation_refresh_likely`
- `semantic_mismatch_possible`
- `artifact_missing`

That would immediately narrow the next step.

#### Proposal 2: add a guided fix command

Example:

```bash
proof trace repair suspect
```

Potential behavior:
- inspect suspect links
- group by likely repair type
- recommend the next action for each group
- optionally execute safe actions like re-reviewing unchanged links

#### Proposal 3: improve first warning text in `audit`

Instead of only:

```text
43 suspect links
```

prefer something closer to:

```text
43 suspect links: linked code changed after review; no missing annotations detected.
Most likely next step: proof trace review --all
Inspect first: proof trace suspect
```

That reduces cognitive branching.

#### Proposal 4: make `trace review --all` output more diagnostic

After review, show not only counts, but also what was actually refreshed:
- reviewed because artifact fingerprint changed
- reviewed because requirement fingerprint changed
- skipped because unresolved
- still suspect after review

That gives the user confidence that the command did the right thing.

## 3. Code MC/DC Can Look Like “Add More Tests” When The Right Move Is “Simplify The Code”

### What happens

The code-side MC/DC reports are useful, but some remaining gaps are not caused by missing behavioral evidence. They are caused by low-value control flow such as:
- tautological loops
- dead or redundant conditions
- audit-hostile branching structure

### Why this is confusing

A user sees a hotspot and naturally thinks:
- “I need another test.”

But in several cases, the better fix is:
- remove a redundant condition
- simplify a loop
- collapse a branch that cannot materially change behavior

### Concrete examples found in this repo

#### Example A: redundant Unicode branch

A condition in `decodeUnicodeEscape` was effectively redundant after the earlier decode step. MC/DC pushed toward an impossible or meaningless proof obligation.

The right fix was code simplification, not more tests.

#### Example B: `for true` in `ArrayEach`

A tautological loop created an MC/DC decision with no behavioral value.

The right fix was to use idiomatic loop structure, not write tests for a meaningless decision.

#### Example C: dead-ish callback gating in `ArrayEach`

Some conditions around callback invocation were only inflating the decision space without representing real meaningful behavior under the current implementation model.

Again, the right move was to simplify.

### Why it matters

This affects whether users trust MC/DC as a quality tool versus experiencing it as bureaucratic pressure.

### Concrete proposals

#### Proposal 1: hotspot output should distinguish likely test gap vs likely code-shape issue

For each hotspot, the CLI should try to label one of these:
- `likely_missing_behavior_test`
- `likely_redundant_condition`
- `likely_tautological_loop`
- `likely_short-circuit-only gap`

Even a heuristic label would help.

#### Proposal 2: add a simplification hint mode

Example:

```bash
proof mcdc report --view hotspots --with-refactor-hints
```

Potential hints:
- “This branch appears tautological under current control flow.”
- “This condition may be structurally redundant after an earlier guard.”
- “Consider splitting parsing and dispatch to reduce compound-decision pressure.”

#### Proposal 3: document the intended philosophy explicitly

There should be a public statement that code-side MC/DC is not only about adding tests. It is also about making decisions explicit, meaningful, and independently testable.

That is important for adoption and for avoiding metric cargo culting.

## 4. It Is Still Not Obvious Which Command Is The Authoritative Next Step

### What happens

The command set is individually strong, but the user often has to infer the correct sequence among:
- `proof workflow check`
- `proof audit`
- `proof trace suspect`
- `proof trace review --all`
- `proof mcdc report --view hotspots`
- `proof trace autolink`

### Why this is confusing

The commands are not redundant, but the CLI still expects the user to know which one is:
- the broad gate
- the diagnostic drill-down
- the repair action
- the confirmation rerun

That is manageable for an expert user, but heavier than it should be for a new user or for a first serious rollout on a repo.

### Concrete example

A realistic flow today can look like this:
- run `proof audit --scope full`
- see a suspect-link warning and MC/DC warning
- run `proof trace suspect`
- decide whether to run `trace review --all`
- run `proof mcdc report --view hotspots`
- decide whether to add tests or simplify code
- rerun `audit`

All of that is logical, but still too much implicit sequencing.

### Concrete proposals

#### Proposal 1: stronger “next command” guidance in failure output

For every failing or warning check, print:
- inspection command
- likely repair command
- confirmation command

Example:

```text
suspect_clean
Inspect: proof trace suspect
Likely repair: proof trace review --all
Confirm: proof audit --scope full
```

#### Proposal 2: add a guided repair command

Example:

```bash
proof fix next
```

Potential behavior:
- inspect current stage/audit failures
- rank the blockers
- suggest a single next command with reason

Example output:

```text
Highest-signal next step: proof mcdc report --view hotspots
Reason: only code_mcdc_coverage remains, and hotspots are available.
```

#### Proposal 3: make command roles explicit in help

For each major command, help text should say clearly whether it is primarily:
- a gate
- a diagnostic
- a repair action
- a report over persisted state

That would reduce the current need to reverse-engineer the command model.

## Suggested Near-Term Roadmap

If I were prioritizing improvements, I would do them in this order:

1. Make shared `.proof/` artifact behavior explicit and safe.
Reason: this causes the most misleading failure modes.

2. Improve suspect-link guidance and repair classification.
Reason: the underlying model is good, but too stateful from the user’s point of view.

3. Improve code MC/DC messaging around simplification vs testing.
Reason: this directly affects whether teams experience MC/DC as useful or adversarial.

4. Strengthen “authoritative next step” guidance across the CLI.
Reason: this reduces the expertise needed to operate the tool well.

## Bottom Line

The remaining issues are now mostly about making the correct mental model obvious.

The CLI is already capable enough to do serious work. The next DX step is to make the operational model legible so users do not have to learn it by tripping over state, sequencing, and interpretation.
