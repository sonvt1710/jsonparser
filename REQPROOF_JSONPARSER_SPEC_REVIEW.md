# Jsonparser ReqProof Spec Review

Date: 2026-04-14
Reviewer stance: NASA-style requirements and verification review
Scope: current stakeholder and system requirements under `specs/`

## Executive Verdict

The stakeholder layer is mostly adequate.

The system layer is materially under-modeled for strong verification.

The main problem is not missing requirement count. The main problem is that nearly every system requirement collapses into a single opaque output boolean such as `key_path_lookup_behavior_is_correct` or `set_behavior_is_correct`. That style is acceptable as a temporary bootstrap, but it is too shallow for strong traceability, meaningful requirement-level MC/DC, or independent verification.

The current spec captures the public API surface reasonably well, but it does not yet capture the public decision structure of that API.

## What Is Good Already

- The stakeholder layer is well partitioned by user-visible capability.
- The top-level API families are covered: lookup, typed access, traversal, multi-path lookup, mutation, unsafe string access, and scalar parsing.
- Stakeholder acceptance criteria are concrete enough to guide decomposition.
- The system layer maps one requirement family to one public helper, which is a workable starting structure.

## What Is Not Good Enough Yet

- System formalization is dominated by umbrella predicates of the form `*_behavior_is_correct`.
- Nominal behavior, not-found behavior, malformed-input behavior, and type-mismatch behavior are often bundled into one requirement.
- Several APIs expose materially richer behavior than the requirement model currently admits.
- Requirement-level MC/DC is weak because a one-boolean requirement has almost no internal decision structure to verify.
- The variable model describes outcomes as conclusions instead of modeling the conditions that drive those outcomes.

## Layer Assessment

### Stakeholder Layer

Verdict: adequate

Why:
The stakeholder requirements are mostly at the right altitude. They describe user-visible capability and intent without collapsing into implementation detail. They should remain relatively concise.

Missing detail:
- A few acceptance criteria could eventually separate nominal behavior from invalid input behavior more sharply, but this is not the main weakness in the spec.

Unnecessary detail:
- None serious at the stakeholder layer.

Recommendation:
- Keep the stakeholder layer mostly stable.
- Drive most refinement work into the system layer, not the stakeholder layer.

### System Layer

Verdict: under-modeled

Why:
The system requirements name the right API surfaces, but their formalization is too coarse to support independent verification. Each requirement currently states that some public behavior is "correct" without modeling the conditions that distinguish success, absence, malformed input, type mismatch, decoding semantics, mutation edge cases, or callback behavior.

Missing detail:
- trigger conditions
- nominal outcomes
- missing-path outcomes
- malformed-input outcomes
- type-compatibility outcomes
- boundary and overflow outcomes
- callback and iteration semantics
- path/index interpretation semantics

Unnecessary detail:
- None in the current text. The problem is lack of behavioral structure, not overspecification.

Recommendation:
- Keep the current API-family grouping as an organizing shell.
- Replace opaque variables with explicit condition and response variables.
- Split requirements where one public API exposes multiple independently testable decisions.

## Requirement Family Review

### STK-REQ-001 to STK-REQ-007

Verdict: adequate

Why:
These requirements express user-visible needs at the correct level. They should remain stable anchors for the system decomposition work.

Recommended action:
- Do not aggressively rewrite these.
- Use them as parent capabilities while decomposing the system layer underneath.

### SYS-REQ-001 `Get`

Verdict: under-modeled

Why:
`Get` is the foundational lookup primitive. The current requirement mixes successful lookup, missing-path behavior, parsing failure, root extraction behavior when no keys are provided, and array-index path semantics into one opaque conclusion.

Missing detail:
- behavior when the addressed path exists
- behavior when the addressed path does not exist
- behavior when input is malformed
- behavior when no keys are provided
- behavior when a path segment is an array index
- definition of returned value slice, type, offset, and error relationship

Recommended decomposition:
- one requirement for successful addressed lookup
- one requirement for missing-path behavior
- one requirement for malformed-input behavior
- one requirement for root-token extraction when no keys are provided
- one requirement for array-index segment handling if that is part of the supported contract

Suggested variable direction:
- `json_input_is_well_formed`
- `path_segments_are_valid`
- `addressed_path_exists`
- `no_keys_are_provided`
- `current_segment_is_array_index`
- `returns_addressed_value_slice`
- `returns_not_exist_type`
- `returns_parse_error`

### SYS-REQ-002 `GetString`

Verdict: under-modeled

Why:
This API combines path lookup, string type enforcement, and decoding semantics. Correctness depends on more than one boolean.

Missing detail:
- addressed value is a JSON string
- escape decoding behavior
- Unicode decoding behavior
- invalid string access behavior
- malformed string encoding behavior

Recommended decomposition:
- one requirement for successful string lookup and decoding
- one requirement for invalid-type or invalid-access behavior
- one requirement for malformed encoded string handling if distinct in the API contract

### SYS-REQ-003 `GetInt`

Verdict: under-modeled

Why:
The API has at least three meaningful verification branches: valid integer retrieval, invalid-type or invalid-access behavior, and malformed numeric handling.

Recommended decomposition:
- valid addressed integer returns `int64`
- invalid access or non-integer addressed value yields error
- malformed numeric representation yields error if distinct from generic invalid access

### SYS-REQ-004 `GetFloat`

Verdict: under-modeled

Why:
Like `GetInt`, this requirement currently hides separate decisions behind one predicate.

Recommended decomposition:
- valid addressed number returns `float64`
- invalid access or incompatible addressed value yields error
- malformed numeric token yields error

### SYS-REQ-005 `GetBoolean`

Verdict: under-modeled

Why:
This API should distinguish valid boolean retrieval from incompatible value or malformed token behavior.

Recommended decomposition:
- valid addressed boolean returns `bool`
- invalid access or incompatible addressed value yields error
- malformed boolean token yields error if behavior is distinct

### SYS-REQ-006 `ArrayEach`

Verdict: under-modeled

Why:
Iteration APIs usually carry multiple externally observable obligations: path resolution, encounter order, callback payload correctness, empty-array behavior, and malformed-input behavior.

Missing detail:
- encounter order guarantee
- callback value and type correctness
- empty-array behavior
- malformed-input behavior
- behavior when the addressed path does not resolve to an array

Recommended decomposition:
- successful iteration over addressed array in encounter order
- callback receives correct element payload and type
- empty array yields defined no-item behavior
- malformed or non-array input yields defined error behavior

### SYS-REQ-007 `ObjectEach`

Verdict: under-modeled

Why:
This API exposes keys, values, and value types. That is already a multi-dimensional contract.

Missing detail:
- object-entry encounter behavior
- key bytes correctness
- value bytes correctness
- value type correctness
- malformed-object behavior
- callback error propagation if part of the public contract

Recommended decomposition:
- successful iteration reports each key/value/type tuple correctly
- malformed object input yields error
- callback error propagation is preserved if the API promises it

### SYS-REQ-008 `EachKey`

Verdict: under-modeled

Why:
This helper has richer observable behavior than the current requirement admits. It resolves multiple paths, identifies which path matched, and defines missing-path behavior per requested path.

Missing detail:
- mapping between callback index and requested path
- behavior for found paths
- behavior for missing paths
- interaction when some paths resolve and others do not
- single-scan efficiency should not be formalized unless you intend to verify performance claims

Recommended decomposition:
- callback index corresponds to the requested path index
- found paths return correct value and type
- unresolved paths return defined missing-path behavior
- mixed found/missing sets preserve per-path correctness independently

### SYS-REQ-009 `Set`

Verdict: under-modeled

Why:
Mutation APIs need sharper contracts than read APIs because callers care about exact edge-case semantics. The current requirement hides update, create, and error behavior in one predicate.

Missing detail:
- updating an existing addressed value
- creating a new addressed value when allowed
- malformed-input behavior
- invalid path behavior
- preservation of surrounding JSON validity after mutation

Recommended decomposition:
- update existing addressed value
- create addressed value where creation semantics are supported
- malformed or unsupported mutation returns error
- output remains valid JSON when mutation succeeds

### SYS-REQ-010 `Delete`

Verdict: under-modeled

Why:
The current text correctly hints that edge cases matter, but the formal model still hides them all behind `delete_behavior_is_correct`.

Missing detail:
- delete existing addressed value
- missing-path behavior
- malformed-input behavior
- deletion from objects versus arrays if behavior differs
- preservation of valid JSON layout after deletion

Recommended decomposition:
- deleting an existing addressed value removes that value
- deleting a missing path yields the defined preserved behavior
- malformed input yields the defined preserved behavior or error
- successful deletion preserves valid JSON structure

### SYS-REQ-011 `GetUnsafeString`

Verdict: mixed

Why:
The core contract is narrower than `GetString`, which is good, but the formal model still hides the meaningful decision points. At the same time, this requirement should not try to formalize memory-allocation internals unless you intend to verify them.

Missing detail:
- addressed lookup success behavior
- raw string mapping behavior without unescaping
- missing-path or invalid-access behavior

Unnecessary detail to avoid:
- GC lifetime explanations
- low-level zero-allocation internals unless made into an explicit measurable nonfunctional requirement

Recommended decomposition:
- successful lookup returns raw string view without JSON unescaping
- missing-path or invalid access yields defined error behavior

### SYS-REQ-012 `ParseBoolean`

Verdict: close, but still under-modeled

Why:
This is one of the simpler APIs, but it still has at least two distinct obligations: correct parsing of valid tokens and deterministic rejection of invalid ones.

Recommended decomposition:
- valid boolean token parses to the corresponding `bool`
- invalid or malformed token yields error

### SYS-REQ-013 `ParseFloat`

Verdict: close, but still under-modeled

Why:
This API is simpler than traversal or mutation, but still benefits from separating valid conversion from malformed rejection.

Recommended decomposition:
- valid floating-point token parses to `float64`
- malformed numeric token yields error

### SYS-REQ-014 `ParseString`

Verdict: under-modeled

Why:
This API combines token validation and decoding semantics, so it needs more than one umbrella variable.

Recommended decomposition:
- valid JSON string token parses to decoded Go string
- escape and Unicode sequences decode correctly
- malformed encoded string yields error

### SYS-REQ-015 `ParseInt`

Verdict: under-modeled

Why:
This API clearly exposes at least three distinct outcomes: valid parse, overflow, and malformed input.

Recommended decomposition:
- valid integer token parses to `int64`
- overflow is detected and surfaced according to the API contract
- malformed integer token yields error

## Priority Order For Decomposition

### Priority 0: Fix the verification shape

Do this first:
- replace umbrella booleans in `specs/system/variables/parser.vars.yaml`
- stop using `*_behavior_is_correct` as the primary formalization pattern
- introduce condition and response variables instead

Why:
Without this, requirement-level MC/DC and formal review will continue to overstate assurance.

### Priority 1: Decompose the most behavior-rich APIs

Do these next:
- `SYS-REQ-001` `Get`
- `SYS-REQ-006` `ArrayEach`
- `SYS-REQ-007` `ObjectEach`
- `SYS-REQ-008` `EachKey`
- `SYS-REQ-009` `Set`
- `SYS-REQ-010` `Delete`

Why:
These functions expose the richest public decision structure and are the biggest sources of spec-to-code mismatch.

### Priority 2: Tighten typed helpers and token parsers

Do these after Priority 1:
- `SYS-REQ-002` through `SYS-REQ-005`
- `SYS-REQ-011` through `SYS-REQ-015`

Why:
They are easier to formalize cleanly once the foundational lookup and mutation semantics are expressed correctly.

## Concrete Example: Better Shape For `SYS-REQ-001`

Current shape:

- `the parser shall always satisfy key_path_lookup_behavior_is_correct`

Recommended shape:

- when the JSON input is well formed and the addressed path exists, the parser shall return the addressed value and its type
- when the JSON input is well formed and the addressed path does not exist, the parser shall report not found
- when the JSON input is malformed before lookup completes, the parser shall report a parsing error
- when no key path is provided, the parser shall return the closest complete JSON value according to the API contract

Better variable style:

- `json_input_is_well_formed`
- `addressed_path_exists`
- `no_key_path_is_provided`
- `returns_addressed_value`
- `returns_not_found`
- `returns_parse_error`

This gives the requirement model real decision structure instead of a post-hoc correctness summary.

## Recommended Next Spec Move

Do not rewrite everything at once.

Instead:

1. Keep the stakeholder layer mostly as-is.
2. Rework `parser.vars.yaml` so variables express conditions and observable outcomes.
3. Fully decompose `SYS-REQ-001` as the pattern-setting example.
4. Apply the same style to `ArrayEach`, `ObjectEach`, `EachKey`, `Set`, and `Delete`.
5. Only then revisit the typed accessors and parse helpers.

## Bottom Line

The current spec is good enough to show API coverage.

It is not yet good enough to support strong independent verification, because the system layer mainly says that public behavior is "correct" instead of modeling the conditions under which different outcomes must occur.

The next improvement should not be "more words". It should be "more decision structure".

## Second Pass: `Get` Family Gap Review

The `Get` family is materially better after decomposing `SYS-REQ-001` into:

- successful addressed lookup
- missing-path behavior
- incomplete-input parse-error behavior
- no-key-path root extraction
- empty-input with key-path behavior

That is the correct first move.

It is still not the full behavior model for `Get`.

### Verdict

Verdict: improved, but still under-modeled

Why:
The current `Get` requirements now cover several top-level observable outcomes, but they still do not cover all of the behavior-distinguishing logic that exists in `Get`, `internalGet`, `searchKeys`, `stringEnd`, `blockEnd`, and the type-detection path. Several important externally visible distinctions are still hidden behind coarse variables such as `addressed_path_exists`, `json_input_is_well_formed`, and `returns_addressed_value_and_type`.

### Missing Logic Still Not Explicitly Modeled

#### 1. Path segment interpretation logic

Current gap:
- `addressed_path_exists` hides multiple distinct behaviors:
  - object key lookup
  - array index lookup
  - invalid array index syntax
  - out-of-bounds array index
  - nested-scope traversal

Why it matters:
- These are not interchangeable internal branches. They change the observable result seen by the caller.

Evidence in tests:
- array index lookup is supported
- malformed array index like `[` or `[]` returns not found
- out-of-scope nested traversal returns not found

Recommended next requirement slices:
- object-key path segments resolve against object members
- array-index path segments resolve against array positions
- malformed array-index path segments yield the defined not-found or invalid-path behavior
- out-of-bounds array indices yield the defined not-found behavior

#### 2. Escaped-key matching semantics

Current gap:
- `searchKeys` and `findKeyStart` explicitly unescape JSON object keys before matching, but the spec does not yet model that as its own obligation.

Why it matters:
- This is caller-visible behavior, not implementation detail.
- A parser that matched only raw encoded key bytes would fail compatibility with the current behavior.

Evidence in tests:
- keys with simple escape sequences are found
- keys with Unicode escapes are found
- keys with surrogate-pair escapes are found

Recommended next requirement slice:
- escaped JSON object keys shall match lookup path segments by their decoded logical key value

#### 3. Returned string-shape semantics for `Get`

Current gap:
- The spec says `Get` returns the addressed value and type, but does not say that string values are returned without surrounding quotes and without JSON unescaping.

Why it matters:
- This is a major externally visible contract distinction between `Get`, `GetString`, and `GetUnsafeString`.

Evidence in tests:
- `Get` strips quotes from string values
- `Get` does not unescape returned string contents
- escaped keys may be decoded for matching even while values remain undecoded

Recommended next requirement slice:
- when the addressed value is a JSON string, `Get` shall return the string token contents without surrounding quotes and without JSON unescaping

#### 4. Result tuple semantics

Current gap:
- The current requirements do not yet define the observable relationship among:
  - returned `value`
  - returned `dataType`
  - returned `offset`
  - returned `err`

Why it matters:
- Two implementations could satisfy the current text while returning different `dataType`/`err`/offset combinations.

Examples that need explicit contract treatment:
- not-found returns `KeyPathNotFoundError` with `NotExist`
- malformed addressed value returns a specific parse-related error
- successful string lookup returns `String` with unquoted bytes
- successful object/array lookup returns slices covering the full nested block

Recommended next requirement slices:
- define observable result tuple for success
- define observable result tuple for not-found
- define observable result tuple for parse-error cases if the exact tuple matters to callers

#### 5. Value-type classification behavior

Current gap:
- `getType` distinguishes string, object, array, boolean, null, number, and unknown value types, but the current `Get` family only models “returns addressed value and type” at a high level.

Why it matters:
- Correct `dataType` classification is part of the public contract, especially because typed helpers build on `Get`.

Recommended next requirement slices:
- addressed string value returns `String`
- addressed object returns `Object`
- addressed array returns `Array`
- addressed boolean returns `Boolean`
- addressed null returns `Null`
- addressed numeric literal returns `Number`
- invalid token shape yields the defined error behavior

#### 6. Malformed-input tolerance policy

Current gap:
- The new decomposition correctly separates empty input from incomplete input.
- It still does not model the current implementation's tolerated malformed-input behavior.

Why it matters:
- The implementation does not enforce one simple rule like “all malformed input yields parse error.”
- Some malformed payloads still return values or not-found results for performance reasons.
- That is externally visible behavior, so a NASA-style review treats it as spec content, not as an implementation accident to ignore.

Evidence in tests:
- some malformed inputs return a successful match
- some malformed inputs return not found
- some malformed inputs return parse error
- comments explicitly state that full malformed-input checking is intentionally not always performed for performance

Recommended next requirement slices:
- explicitly define which malformed-input classes must produce parse error
- explicitly define which malformed-input classes may still produce successful lookup or not-found if the parser can isolate the addressed token

If you do not want to commit to tolerant malformed-input behavior as a stable contract, then the spec should instead mark those tests as implementation tolerance rather than requirement evidence.

#### 7. Empty-path root extraction boundaries

Current gap:
- `SYS-REQ-018` covers well-formed no-key-path extraction in general, but does not yet separate:
  - object root extraction
  - array root extraction
  - scalar root extraction
  - malformed no-key-path input

Why it matters:
- These are different observable outcomes and may require different tests and edge-case handling.

Recommended next requirement slices:
- no-key-path object extraction
- no-key-path array extraction
- no-key-path scalar extraction if supported
- malformed no-key-path input behavior

#### 8. Nested-scope and sibling-scope matching behavior

Current gap:
- The code in `searchKeys` tracks nesting level and only matches keys at the intended scope, but the current requirements do not explicitly state that lookup must honor JSON structure rather than substring coincidence.

Why it matters:
- This is central to correctness.
- Without it, an implementation could match the wrong key at the wrong depth and still plausibly claim to return “a value.”

Evidence in tests:
- applying scope to nested paths
- avoiding sibling leakage
- object-in-array path traversal cases

Recommended next requirement slice:
- key-path lookup shall respect JSON structural scope and shall not satisfy a deeper or sibling path segment from an unrelated scope

#### 9. Structural-block completeness for arrays and objects

Current gap:
- `blockEnd` governs whether objects and arrays are considered complete enough to return.
- The current requirements do not yet say when unterminated arrays/objects must yield parse error versus when partial lookup is still acceptable.

Why it matters:
- This is externally visible and already appears in tests.

Recommended next requirement slices:
- addressed array values require a complete closing `]` when the contract demands full array extraction
- addressed object values require a complete closing `}` when the contract demands full object extraction
- distinguish these from malformed but tolerated cases if that tolerance is intentional

### What Does Not Need Promotion Yet

These branches are currently better treated as implementation detail unless a stronger assurance goal emerges:

- exact loop structure in `searchKeys`
- stack-buffer versus heap-buffer choices during unescape
- helper call ordering such as `nextToken` before `getType`
- internal scan mechanics that do not change the required externally visible outcome

### Next `Get` Decomposition Priorities

If continuing the `Get` family before moving to other APIs, the next highest-value additions are:

1. escaped-key matching semantics
2. path-segment interpretation including array-index behavior
3. returned string-shape semantics for `Get`
4. nested-scope matching behavior
5. malformed-input tolerance policy
6. result tuple semantics for `value`, `dataType`, `offset`, and `err`

### Bottom Line For `Get`

The `Get` family is no longer opaque, which is good.

It still does not cover all behavior-changing logic in the function and its helper stack.

The missing logic is not “every branch in the code.” The missing logic is the set of remaining branches that change what a caller can observe or rely on.
