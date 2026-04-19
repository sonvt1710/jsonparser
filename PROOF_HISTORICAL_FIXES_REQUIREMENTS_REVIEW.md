# Proof Historical Fixes Requirements Review

Date: 2026-04-14
Author: Codex
Scope: Review of recent and historically important merged pull requests to identify what requirement gaps they reveal, with emphasis on the delete-related security fix merged on 2026-03-19.

## Purpose

Translate historical bug fixes into explicit requirement changes instead of leaving them as test-only tribal knowledge.

## Reviewed Merges

Recent merges checked from local git history:

- `a69e7e0` on 2026-03-19: merge of PR #276, `fix: prevent panic on negative slice index in Delete with malformed JSON (GO-2026-4514)`
- `61b32cf` on 2022-04-18: merge of PR #241, `fix EachKey pIdxFlags allocation`
- `2181e83` on 2022-04-18: merge of PR #244, CI update
- `a6f867e` on 2021-11-25: merge of PR #239, `Fuzzing: Add CIFuzz`
- `dc92d69` on 2021-06-20: merge of PR #228, null handling for typed getters
- `2d9d634` on 2021-06-20: merge of PR #231, ParseInt overflow check fix
- `df3ea76` on 2021-01-08: merge of PR #221, labeled `CVE-2020-35381`

## Important Correction

The older CVE-era merge on 2021-01-08 was not a `Delete` bug.

The underlying change was commit `1e1db9e`, `handle "[" as a malformed array index by returning NotFound`, which hardened `Get`/`searchKeys` path interpretation for malformed array-index syntax.

The delete-specific security fix was the later merge on 2026-03-19 for `GO-2026-4514`.

## What The 2026 Delete Security Fix Actually Changed

The `Delete` implementation used to assume certain intermediate lookup operations either succeeded or only failed with `KeyPathNotFoundError`.
That assumption was false for malformed or truncated input.
In those cases, `Delete` could continue using invalid offsets and panic.

The 2026 fix changed `Delete` to return the original input early on any intermediate lookup/parsing error instead of continuing into offset arithmetic.

Relevant current implementation area:
- `parser.go`, `Delete`, around lines 733-806

Relevant regression tests added for this issue:
- `parser_test.go`, around lines 234-249
- malformed JSON without enclosing braces should not panic
- malformed JSON with truncated value should not panic
- malformed nested JSON with truncated value should not panic

## Requirement Gap Exposed By The 2026 Fix

Current requirement `SYS-REQ-010` is too coarse.
It mixes together:
- no-path behavior
- successful deletion behavior
- missing-target behavior
- malformed/unusable-input behavior

That broad requirement does mention preserving input when the input is unusable for deletion, but it does not explicitly state the safety property that matters most for a security fix:

- `Delete` shall not panic on malformed or truncated input

That omission is important.
A regression test can catch the known cases, but the requirement should explicitly classify panic-freedom under unusable input as part of the contract.

## Requirement Gap Exposed By The 2021 CVE-Era Fix

The older CVE-era fix was about malformed array-index path syntax in lookup, not deletion.
This requirement gap is already modeled much better in the current spec via:
- `SYS-REQ-022`, malformed array-index syntax returns the defined not-found result

That is a good example of the right pattern:
- one distinct externally visible edge case
- one explicit requirement row
- not hidden inside a generic umbrella requirement

## Recommended Requirement Changes

### 1. Tighten The Stakeholder Requirement

Current stakeholder acceptance criterion `STK-REQ-005` AC-2 says:
- Delete returns the expected mutated payload or preserved edge-case behavior when paths are missing or input is malformed

That should be made more explicit.

Recommended direction:
- mention malformed, truncated, or unusable input explicitly
- mention non-crashing behavior explicitly

Example revised acceptance criterion text:
- A caller can delete an addressed JSON value through `Delete` and receive either the expected mutated payload or the unchanged original payload for missing, malformed, truncated, or otherwise unusable input, without process crash or panic.

### 2. Split `SYS-REQ-010` Into Smaller Requirement Rows

`SYS-REQ-010` should not carry all of `Delete` alone.
It should be decomposed into at least these rows:

- no-path behavior
- successful deletion behavior
- missing-target preservation behavior
- unusable-input robustness behavior

This is the same decomposition logic already used successfully elsewhere in the spec.

### 3. Add An Explicit Delete Robustness Requirement

Recommended new system requirement concept:

- When `Delete` receives malformed, truncated, or otherwise unusable input for the requested path resolution, it shall return the original byte payload unchanged and shall not panic.

Suggested variables:
- `delete_input_is_unusable_for_requested_path`
- `delete_returns_original_input_on_unusable_input`
- `delete_completes_without_panic`

Suggested FRET-style form:
- the parser shall always satisfy !delete_input_is_unusable_for_requested_path | (delete_returns_original_input_on_unusable_input & delete_completes_without_panic)

This is the requirement that directly addresses the 2026 security bug class.

### 4. Keep Successful Deletion Separate From Robustness

The mutation success case should remain independent:

Suggested concept:
- When the addressed delete target exists and is isolatable, `Delete` shall return the JSON document with that target removed.

This avoids overloading one requirement with both happy-path mutation and robustness semantics.

### 5. Separate Missing-Target From Malformed-Input

The current requirement treats these together:
- target missing
- input unusable

Those are not the same external behavior class.
They may both preserve input today, but they arise from different causes and should be independently reviewable.

Recommended concept:
- When the addressed target does not exist in otherwise usable input, `Delete` shall return the original byte payload unchanged.

### 6. Consider A Separate Non-Corruption Requirement

Historical `Delete` history also includes fixes like:
- `7c4dc07` `Original data should not be corrupted on a delete (#166)`

That suggests another externally visible invariant worth specifying:
- `Delete` shall not corrupt the caller's original input buffer while producing the returned result.

Whether to formalize that depends on whether you want to treat input-buffer preservation as contractual API behavior or only as an implementation detail.
For a byte-slice API in Go, I would lean toward treating it as contractual.

## Suggested Spec Shape

A stronger `Delete` requirement family would look more like this:

- `Delete` with no path returns an empty payload.
- `Delete` with an existing addressed target returns the payload with that target removed.
- `Delete` with a missing addressed target in otherwise usable input returns the original payload unchanged.
- `Delete` with malformed, truncated, or otherwise unusable input returns the original payload unchanged.
- `Delete` shall not panic during any of the above cases.
- optionally: `Delete` shall not corrupt the caller's original input buffer.

## Why This Is Better

This change does three things:

- makes the security-relevant property explicit instead of accidental
- makes review easier because malformed-input robustness is no longer buried in a broad OR-expression
- turns historical bug fixes into requirement rows that future tools and agents can reason about directly

## Process Rule To Generalize

For historical fixes, the rule should be:

- if a bug changed externally visible behavior, create or refine a behavior requirement
- if a bug was a panic, crash, hang, overflow, or corruption issue, create or refine a robustness/safety requirement
- if a bug only exposed redundant or unclear internal logic, fix code and tests without necessarily expanding the external contract

That is how old PR history becomes an assurance asset rather than just archaeology.

## Bottom Line

The 2026 delete security fix should drive a new explicit `Delete` robustness requirement, not just more tests under the existing umbrella requirement.

The 2021 CVE-era fix shows the right decomposition pattern already: model the edge case as a first-class requirement row instead of burying it inside a generic lookup contract.
