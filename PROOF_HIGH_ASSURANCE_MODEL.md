# Proof High-Assurance Model

Date: 2026-04-14
Author: Codex
Scope: How `Proof` should think about assurance if it borrows methods from aerospace, automotive, NASA-style, and other high-assurance engineering cultures.

## Purpose

This note answers a product question, not just a project question:

- What kind of claim can `Proof` help a team make?
- What is deterministic and rule-based?
- What still requires human judgment?
- How should `Proof` structure the path from ordinary engineering toward stronger assurance?

## Core Position

High-assurance organizations do not try to prove that a program is “bug-free” in an unlimited sense.

They instead build a bounded and defensible claim such as:

- within a declared operational scope
- for a declared set of hazards, behaviors, and failure conditions
- the implementation satisfies its requirements
- with evidence commensurate to the consequence of failure

That is the right mental model for `Proof`.

## What `Proof` Should Help Teams Say

Weak claim:
- “tests pass”

Better claim:
- “the implementation, tests, and traced requirements are consistent under the configured verification policy”

High-assurance style claim:
- “within the declared supported behavior and failure model, all identified externally visible behavior classes are specified, traced, and verified with diverse evidence”

`Proof` should be built to support the third kind of claim.

## The Assurance Ladder

### Level 1: Trace-Complete

Meaning:
- requirements exist
- requirements are linked to code, tests, and docs
- basic checks pass

What `Proof` can enforce well:
- trace completeness
- annotation validity
- missing evidence checks
- review freshness

### Level 2: Behavior-Complete

Meaning:
- public behavior has been decomposed into externally visible requirement rows
- not just nominal success behavior, but also edge and failure behavior

Examples of obligation classes:
- nominal case
- empty input
- missing field/path
- malformed structure
- incomplete input
- invalid escape or token class
- wrong type
- boundary values
- invariants and preservation properties
- unsupported behavior declaration

This is where requirements become meaningfully complete instead of merely present.

### Level 3: Evidence-Diverse

Meaning:
- requirements are not checked only with example tests
- multiple evidence classes exist

Evidence classes include:
- example-based tests
- boundary-value tests
- malformed-input class tests
- property-based tests
- fuzzing
- differential tests
- regression corpora
- structural coverage / MC/DC
- formal invariants where justified

This is the first major leap in confidence beyond ordinary test suites.

### Level 4: Robustness-Qualified

Meaning:
- the system is not just conformant in normal cases, but robust under adversarial, malformed, or degenerate inputs

Evidence includes:
- persistent fuzzing
- regression corpus preservation
- sanitizer/race tooling where applicable
- panic-freedom / non-crash expectations
- performance/resource-limit checks where required

### Level 5: Formally-Constrained Critical Properties

Meaning:
- some critical semantic claims are not only empirically tested, but analytically or formally justified

Examples:
- key parser invariants
- mutation preservation properties
- decision partition correctness
- safety-critical transformation invariants

This is where `Proof` begins to move from strong engineering discipline toward genuine high-assurance verification support.

## Deterministic vs Judgment-Based Work

The process should be explicitly split.

### Deterministic, Rule-Based Work

These are appropriate for tool/agent enforcement:
- schema and syntax validity
- trace completeness
- stale evidence detection
- review freshness checks
- coverage thresholds
- code MC/DC measurement
- requirement-template completeness
- obligation checklist completeness when answers are explicit
- artifact freshness and reproducibility checks

This category should be highly automated.

### Structured Judgment

These are not arbitrary, but they require human semantic decisions:
- what behaviors are in scope
- whether a distinction is externally visible or merely internal detail
- what unsupported behavior should be explicitly rejected versus left undefined
- what constitutes a meaningful requirement decomposition
- what residual risk is acceptable
- what level of evidence is sufficient for a release or approval claim

The right model is:
- deterministic tooling for enforcement
- human judgment for semantics, scope, and risk acceptance

## How High-Assurance Teams Would Attack A Parser Problem

A high-assurance team would not start by asking:
- “How do we prove the parser has no bugs?”

They would ask:
- “What exact assurance claim do we want to make about parser behavior?”

Then they would work through these steps.

### Step 1: Define Operational Scope

For a parser-like library, this means clarifying:
- what inputs are supported
- what malformed inputs are rejected
- whether recovery or best-effort behavior is normative or incidental
- what duplicate-key policy is
- what error classes are promised
- what mutation operations must preserve
- what performance/resource assumptions matter

Anything not classified as supported, rejected, or explicitly undefined weakens the assurance case.

### Step 2: Enumerate Externally Visible Behavior Units

Not code units. Behavior units.

Examples:
- returns addressed value when path exists
- returns not-found when path absent
- rejects malformed array index syntax
- decodes escaped string in decoded getter path
- preserves raw token form in raw getter path
- rejects malformed escapes
- `Delete` preserves unrelated content
- `Set` preserves structure except targeted mutation
- traversal APIs do not panic on malformed inputs

This list should be human-curated.

### Step 3: Apply A Fixed Obligation Checklist To Each Behavior Unit

For each behavior unit, require explicit answers for:
- nominal behavior
- empty input
- incomplete input
- malformed input class(es)
- wrong type
- boundary values
- invariants / preservation
- unsupported scope declaration

If a behavior unit does not answer those where applicable, it is incomplete.

### Step 4: Convert Obligation Answers Into Requirement Rows

One vague requirement should become multiple precise requirement rows.

Example transformation:
- from: “Get returns the value at a key”
- to:
  - returns addressed value for existing path in well-formed input
  - returns not-found for absent path in well-formed input
  - returns parse-related error for incomplete addressed token
  - rejects malformed array-index syntax
  - preserves raw string escape form in raw API path
  - decodes escapes in decoded API path

This is the main mechanism for making semantics complete.

### Step 5: Attach Diverse Evidence To Each Requirement Row

A good requirement row should be backed by one or more of:
- direct example tests
- boundary or malformed-input tests
- property-based tests
- fuzz reachability or corpus evidence
- differential checks
- formal analysis where required

### Step 6: Use Structural Coverage As A Backstop, Not The Primary Goal

This is standard high-assurance logic.

Requirements-based testing comes first.
Structural coverage is then used to reveal:
- missing tests
- missing requirements
- dead code
- redundant conditions
- coupled logic that is hard to justify independently

In other words:
- MC/DC should reveal assurance debt
- not become a pure metric game

### Step 7: Track Anomalies Systematically

When a bug is found, the process should ask:
- which requirement was missing or weak?
- which evidence class should have found this?
- what neighboring defect class might also exist?
- what permanent regression artifact should be added?

That is how assurance matures over time.

## What `Proof` Should Encode As Product Rules

### Rule 1: Every Public Behavior Must Be Classified

For each behavior unit, `Proof` should encourage or enforce classification as:
- supported
- rejected
- undefined / intentionally unsupported

### Rule 2: Every Requirement Family Must Cover More Than Nominal Success

For critical APIs, `Proof` should support obligation templates that force teams to consider:
- empty
- incomplete
- malformed
- boundary
- type mismatch
- preservation / invariant

### Rule 3: Evidence Should Be Typed, Not Just Present

Evidence should not be merely “a test exists.”
It should be typed as:
- example
- boundary
- malformed-input
- property
- fuzz
- differential
- review
- formal

This lets teams set policy like:
- each critical requirement needs at least 2 evidence classes
- each parser requirement family needs at least 1 generative evidence class

### Rule 4: Structural Coverage Findings Should Be Triaged Into Classes

When MC/DC or coverage finds a gap, `Proof` should help classify it as one of:
- missing behavior test
- missing requirement row
- likely redundant condition
- likely dead/tautological code
- likely unsupported behavior gap

This is much better than treating every hotspot as “write another test.”

### Rule 5: Claims Must Be Bounded

`Proof` should encourage bounded claims such as:
- “within supported parser behavior X...”
- “for declared malformed-input classes Y...”
- “under configured verification policy Z...”

It should never encourage unlimited claims like “bug-free.”

## What Remains Human-Only

Even in a strong system, humans must still decide:
- what the supported scope is
- what hazards matter
- what risk level applies
- what ambiguity means semantically
- what unsupported behavior is acceptable
- when evidence is sufficient for a release claim
- when residual risk is accepted

That is not a weakness. That is correct assurance engineering.

## Product Implication For `Proof`

`Proof` should not position itself as:
- a magical correctness prover
- a pure static checker
- a requirements linter with dashboards

It should position itself as:
- a system for making engineering claims structured, reviewable, auditable, and increasingly automatable

That is a strong product story.

## Recommended Next Product Direction

If `Proof` wants to move closer to high-assurance usefulness, the next capabilities should be:

1. obligation-template support for requirement completeness
2. typed evidence classes
3. stronger unsupported/supported/undefined behavior modeling
4. better hotspot classification for “test gap vs code-shape issue”
5. better support for independent evidence classes
6. clearer claim language and release/approval semantics

## Bottom Line

The aerospace / automotive / NASA-style answer is not:
- “make the tool prove bug-free”

It is:
- “make the tool help teams build bounded assurance claims, backed by structured evidence, under explicit policy, with human ownership of semantics and risk”

That is the right long-term intellectual model for `Proof`.
