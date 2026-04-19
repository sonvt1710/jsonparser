# Proof CLI UX Log

Date: 2026-04-14
Evaluator: Codex
Scope: CLI-only validation of `proof` against scratch copies of `jsonparser`
Current validation repo: `/Users/leonidbugaev/go/src/jsonparser`

## Goal

Track only current active UX issues found while dogfooding `proof` through its CLI on a non-`proof` project.

## Active Issues

None currently confirmed on the installed CLI after the latest revalidation pass.

## Resolved Since Earlier Passes

### 1. `workflow check --stage spec` now agrees with standalone `proof validate` on the malformed acceptance-criteria repro

- Revalidated on the current CLI in a scratch copy by removing `text` and `testable` from `stakeholder.acceptance_criteria[0]` in `STK-REQ-001`.
- Current behavior:
  - `proof workflow check --stage spec --verbose` now fails `validate_passes`
  - it also flags the same structural acceptance-criteria issues under `stakeholder_acceptance_criteria` and `l0_stakeholder_complete`
  - `proof validate` reports the same requirement error
- This should no longer be tracked as an active UX issue.

### 2. `trace autolink` help no longer promotes deprecated annotation syntax as the primary form

- Current `proof trace autolink --help` now documents:
  - preferred production form: `// SYS-REQ-042`
  - preferred test form: `// Verifies: SYS-REQ-042`
  - compatibility syntax separately under `Supported compatibility syntax`
- `proof help req_impl_coverage` is also aligned with the preferred forms and explicitly labels old `reqproof:*` comments as compatibility-only.
- This should no longer be tracked as an active UX issue.

### 3. `proof audit --scope full` now shows verify-phase progress on the current CLI

- Revalidated on the current CLI by comparing `proof audit --scope full` with `proof workflow check --stage verify --verbose`.
- `audit` now emits detailed verify-phase progress lines such as:
  - `verify validate: started`
  - `verify realize: started`
  - downstream verify-step status lines through completion
- The older complaint about long silent verify periods did not reproduce on the current binary.

### 4. `proof help test_mcdc_annotations` now resolves correctly

- Revalidated on the current CLI with:
  - `proof help test_mcdc_annotations`
- Current behavior:
  - the config-key form now resolves and shows the `Test MC/DC Annotations Clean` help topic
- This should no longer be tracked as an active UX issue.

### 5. `proof coverage link` no longer treats the truncated coverprofile repro as a successful empty report

- Revalidated on the current CLI in a scratch copy of `jsonparser` with:
  - a full profile at `/tmp/jsonparser-full.coverprofile`
  - a deliberately truncated profile at `/tmp/jsonparser-truncated.coverprofile`
- Current behavior:
  - the truncated profile no longer produces a fake successful empty summary
  - it now fails instead of silently returning `0 requirements linked`
- This should no longer be tracked as an active UX issue.
