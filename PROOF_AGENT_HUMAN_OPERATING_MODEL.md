# Proof Agent/Human Operating Model

Date: 2026-04-14
Author: Codex
Scope: Product-design guidance for how `Proof`, agents, and humans should divide work in an assurance workflow.

## Purpose

This note answers a core product question:

- What should the tool do automatically?
- What should an agent be allowed to decide or propose?
- What must remain explicitly human?
- What should the workflow stages and approval model look like?

## Core Principle

Machines should produce:
- evidence
- analysis
- proposals
- diagnostics
- confidence estimates

Humans should own:
- meaning
- scope
- risk acceptance
- normative approval
- release claims

Short form:
- machines produce evidence and recommendations
- humans approve semantics, risk, and claims

This should be the central product philosophy for `Proof`.

## Three Classes Of Work

### 1. Deterministic Machine Work

These actions should be fully automatable and policy-enforced.

Examples:
- schema validation
- requirement structure checks
- trace completeness checks
- stale-link detection
- artifact freshness checks
- coverage and MC/DC measurement
- report generation
- checklist enforcement where answers are explicit
- evidence collation

Here the agent/tool should simply do the work.
No human should be required unless policy explicitly says otherwise.

### 2. Bounded Judgment With Machine Proposal

These are cases where the agent can do most of the work but should not silently make the final normative call.

Examples:
- likely incomplete requirement detection
- suspect-link triage
- “test gap vs refactor gap” hotspot triage
- implementation-to-requirement candidate matching
- whether a test block is a true requirement witness or merely incidental coverage
- likely supported / unsupported behavior gap detection

Here the agent should output:
- recommendation
- evidence summary
- rationale
- confidence
- alternatives when plausible
- explicit escalation reason if policy requires approval

### 3. Human-Only Normative Decisions

These decisions should remain explicitly human.

Examples:
- approving requirements
- approving waivers or exemptions
- deciding whether a behavior is supported, rejected, or undefined
- accepting residual risk
- resolving ambiguous semantics
- approving release/readiness claims
- deciding whether evidence is sufficient for a critical requirement family

The agent may prepare the review packet, but should not silently finalize these decisions.

## Simple Decision Rule

A strong product rule is:

- if an action changes facts, the machine may do it
- if an action changes claims, scope, responsibility, or accepted risk, a human must approve it

Examples:
- machine can measure MC/DC
- machine can refresh traceability links
- machine can generate review packets
- human must approve a requirement as normative
- human must approve a waiver
- human must approve a release-scope claim

This rule is simple, defensible, and scalable.

## What The Workflow Should Model

The workflow should distinguish between:
- machine readiness
- human approval

Those are not the same thing.

### Suggested Stages

#### `draft`
Meaning:
- content exists but is not yet trusted
- agent may create or refine
- no human approval required

#### `analyzed`
Meaning:
- machine checks have run
- trace and evidence candidates exist
- gaps and ambiguities are identified

#### `review_required`
Meaning:
- the item is blocked on human normative judgment
- the agent should prepare a concise review packet

#### `approved`
Meaning:
- a human explicitly approved the current version/fingerprint
- approval should be tied to comment, approver, and revision

#### `implemented`
Meaning:
- implementation evidence exists and machine gates pass

#### `verified`
Meaning:
- verification evidence exists and configured gates pass

#### `accepted` or `released`
Meaning:
- a human owner accepted the assurance posture for use/release

This separation avoids conflating “machine checks passed” with “business/engineering acceptance happened.”

## Approval Should Be First-Class

Approvals should not be loose comments if `Proof` is meant to support serious engineering governance.

Every approval should record:
- object approved
- exact revision or semantic fingerprint
- approver identity/role
- timestamp
- mandatory rationale/comment where policy requires it
- approval scope

Examples of approval scope:
- semantics approved
- waiver approved
- review refresh acknowledged
- release claim accepted

## Where Explicit Human Push/Approve Is The Right Pattern

The explicit review/approve flow is the right pattern when a transition changes the project’s assurance claim.

Good candidates:
- requirement approval
- waiver approval
- ambiguity resolution
- scope decision
- residual-risk acceptance
- critical evidence sufficiency approval

Poor candidates:
- every trace refresh
- every generated report
- every low-risk docs update
- every auto-linked artifact relationship

If approval is required too often, users will rubber-stamp.
If it is reserved for claim-bearing transitions, it stays meaningful.

## How The Agent Should Decide Whether To Escalate

The agent should not guess ad hoc. `Proof` should give it policy-backed escalation rules.

### Agent may act autonomously when
- the action is deterministic under project policy
- the action does not alter accepted semantics
- the action does not create or approve a waiver
- the action does not accept residual risk
- the evidence clearly supports one outcome
- rollback is easy and non-normative

### Agent should escalate when
- multiple semantic interpretations are plausible
- requirement meaning is ambiguous
- scope boundary is unclear
- an exemption or waiver is being introduced
- a policy requires human approval
- residual risk must be accepted
- evidence is conflicting
- the proposed action would alter normative system behavior or compliance posture

## What The Agent Should Produce Before Escalating

When escalation is required, the agent should not just stop and ask a vague question.
It should prepare a review packet.

A good review packet includes:
- object under review
- what changed
- why escalation is required
- current evidence summary
- risks / unresolved ambiguity
- recommended decision
- confidence and alternatives

This keeps the human focused on decision quality instead of information gathering.

## Recommended Product Feature: Decision Policy Model

`Proof` should model not only checks, but also decision classes.

For each object or transition, policy should define things like:
- approval required or not
- who may approve
- whether the agent may propose
- whether the agent may finalize
- whether comment/rationale is required

Conceptually, something like:

```yaml
policy:
  approval:
    required: true
    role: system_owner
    comment_required: true
  agent:
    may_propose: true
    may_finalize: false
```

That would make the agent/human boundary explicit, auditable, and configurable.

## Recommended Product Feature: Review Packets

Example command:

```bash
proof review prepare <object>
```

Desired output:
- what is being reviewed
- current status and evidence
- unresolved ambiguity
- recommended action
- exact fingerprint/revision to approve

This turns review from “read everything manually” into “make a bounded decision.”

## Recommended Product Feature: Confidence + Reason Taxonomy

The agent should not just say “I need a human.”
It should say why.

Examples:
- ambiguous semantics
- unsupported behavior decision required
- waiver/exemption required
- conflicting evidence
- approval policy requires human signoff
- scope boundary unclear
- risk acceptance required

This makes escalation legible and trustworthy.

## Business Framing For `Proof`

From a product/business perspective, `Proof` should be framed as:

- an assurance operating system for engineering teams and agents

Its value is:
- reducing assurance labor
- structuring evidence
- making agent work governable
- increasing accountability without slowing work unnecessarily
- making review and approval explicit where it matters

That is stronger and more differentiated than being seen as only a traceability or compliance tool.

## Anti-Patterns To Avoid

### Anti-pattern 1: human as rubber stamp

If the agent does everything and the human only clicks approve, the approval loses meaning.

### Anti-pattern 2: human as evidence assembler

If the tool finds issues but the human must manually reconstruct all meaning and evidence, the system does not scale.

The right middle ground is:
- agent builds the case
- human approves the claim

## Practical Product Rule Set

If I had to reduce this to a short operational policy, it would be:

1. Machine checks should be default-on for deterministic evidence work.
2. Agents may prepare, correlate, and recommend.
3. Humans must approve semantics, waivers, scope, and risk.
4. Every approval must bind to an exact reviewed state.
5. Escalation must include a reason and a prepared review packet.
6. Workflow stages must distinguish readiness from acceptance.

## Bottom Line

The right role split is not:
- “replace the human”

and not:
- “force the human to do everything important manually”

It is:
- `Proof` enforces structure
- agents do scalable evidence and analysis work
- humans make the bounded set of normative decisions that actually require judgment and accountability

That is the operating model I would design the product around.
