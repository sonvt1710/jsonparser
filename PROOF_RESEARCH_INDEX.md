# Proof Research Index

Date: 2026-04-14
Author: Codex
Scope: Index of research and design notes created while dogfooding `Proof` on `jsonparser` and thinking about `Proof` as a product.

## Purpose

Keep the research set structured, discoverable, and readable in a sensible order.

## Research Structure

This research set is intentionally split into three layers:

- strategy and product model
- active DX/UX concerns from real dogfooding
- project-specific spec and verification analysis

That separation matters because these are different kinds of knowledge:

- strategy documents explain what `Proof` should be as a product
- DX/UX notes capture what felt confusing while actually using the CLI
- project-specific notes record what we learned by applying the method to `jsonparser`

## Core Conclusions

If someone reads only one page, these are the main takeaways:

- `Proof` should help teams make bounded assurance claims, not unlimited “bug-free” claims.
- deterministic checks and evidence gathering should be machine-driven by default.
- semantics, scope, waivers, and risk acceptance should remain explicitly human-owned.
- requirements need to model externally visible behavior classes, not just nominal success paths.
- MC/DC and coverage are backstops that expose assurance debt; they should not become the primary goal.
- agent workflows should escalate with clear reasons and prepared review packets, not vague requests for help.

## Recommended Reading Order

### 1. Product / Strategy

- [PROOF_HIGH_ASSURANCE_MODEL.md](/Users/leonidbugaev/go/src/jsonparser/PROOF_HIGH_ASSURANCE_MODEL.md)
  What a high-assurance framing for `Proof` should look like. Covers bounded claims, evidence classes, deterministic policy vs human judgment, and the assurance ladder.

- [PROOF_AGENT_HUMAN_OPERATING_MODEL.md](/Users/leonidbugaev/go/src/jsonparser/PROOF_AGENT_HUMAN_OPERATING_MODEL.md)
  How `Proof`, agents, and humans should divide work. Covers workflow stages, approval boundaries, escalation rules, and why “machine produces evidence, human approves claims” is the right model.

### 2. Active DX / UX Product Concerns

- [PROOF_CLI_ACTIVE_DX_CONCERNS.md](/Users/leonidbugaev/go/src/jsonparser/PROOF_CLI_ACTIVE_DX_CONCERNS.md)
  Only the still-active confusing parts after the latest validation pass, with concrete examples and concrete proposals.

- [proof-ux-log.md](/Users/leonidbugaev/go/src/jsonparser/proof-ux-log.md)
  Current CLI-only UX log, intentionally trimmed to current active issues and validated resolved items.

### 3. Current Project-Specific Spec Research

- [REQPROOF_JSONPARSER_SPEC_REVIEW.md](/Users/leonidbugaev/go/src/jsonparser/REQPROOF_JSONPARSER_SPEC_REVIEW.md)
  Deep review of `jsonparser` specs using a NASA-style verification lens. Covers what was strong, what was under-modeled, and how requirement families should be decomposed.

- [REQPROOF_NASA_SPEC_PROMPT.md](/Users/leonidbugaev/go/src/jsonparser/REQPROOF_NASA_SPEC_PROMPT.md)
  The reusable prompt that captures the NASA-style review stance.

## What Each Document Is For

### If the question is strategic
Read:
- `PROOF_HIGH_ASSURANCE_MODEL.md`
- `PROOF_AGENT_HUMAN_OPERATING_MODEL.md`

### If the question is “what still feels confusing in the CLI?”
Read:
- `PROOF_CLI_ACTIVE_DX_CONCERNS.md`
- `proof-ux-log.md`

### If the question is “how deep should specs go on a real project?”
Read:
- `REQPROOF_JSONPARSER_SPEC_REVIEW.md`
- `REQPROOF_NASA_SPEC_PROMPT.md`

## Suggested Maintenance Rule

Keep this research set current with a simple rule:

- update the strategy docs when the product philosophy changes
- update the DX/UX docs when dogfooding reveals new active confusion
- update the project-specific notes when the `jsonparser` assurance model or spec depth changes

Resolved historical issues should not accumulate in the active DX/UX files unless they still matter as product lessons.

## Current State Of The Project Used For Dogfooding

At the time of this index:
- `go test ./...` passes
- `proof workflow check --stage implement --verbose` passes
- `proof audit --scope full` passes with `Errors: 0, Warnings: 0`

This matters because the research was produced through a real end-to-end dogfooding cycle, not only by reading docs.

## Bottom Line

The repo now has a real research entry point instead of scattered chat history.
Start here, then branch into:

- product philosophy
- agent/human workflow design
- current CLI concerns
- project-specific verification depth

That should make the work reusable and auditable instead of conversationally transient.
