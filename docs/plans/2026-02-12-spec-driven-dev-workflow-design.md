# Spec-Driven Development Workflow Design

> How AI agents develop from specs with iterative feedback loops and quality verification

**Version:** 2.0 (post senior review)
**Date:** 2026-02-12
**Reviewers:** BE Senior, Web Senior, QA Lead

---

## Motivation

In human software development, specs (PRD, Figma, etc.) are living documents. Developers read a spec, start coding, discover ambiguities, push back with questions, and the spec evolves through the process. QA then tests against the spec and may find both code bugs and spec gaps.

woosgem-mda's core principle is "specs are the source, code is derived." But generating code from a static spec produces unknown quality. This design introduces a **human-like iterative development workflow for AI agents** — where agents don't just implement specs, they critique, question, and improve them through the development process.

---

## Prerequisite: Layer Model Reality

> Addresses: terminology mismatch (BE-F1, WS-F001), missing Layer 1 (BE-F2, WS-F003/F009)

### The Gap

The schema system defines a clean 3-layer architecture:

| Layer | File | Purpose | Vocabulary |
|-------|------|---------|------------|
| Layer 0 | `spec.yaml` | Platform-agnostic | properties, slots, behaviors, semantics.role |
| Layer 1 | `spec.web.yaml` | Web platform mapping | HTML tag, ARIA, CSS hooks, DOM events |
| Layer 2 | `spec.react.yaml` | Framework patterns | JSX, hooks, props, Context |

But the actual `spec.yaml` files use **web platform terms**: `attributes` (not properties), `content` (not slots), `interactions` (not behaviors), `semantics.tag: button` (HTML), `visual_mapping.class: btn` (CSS), `accessibility.aria` (WAI-ARIA). Layer 0 and Layer 1 are effectively merged.

### Decision: Accept the Merge, Document It

For this workflow, we accept the current reality:

- **`spec.yaml` = Layer 0 + Layer 1 merged for web components.** It defines both what the component IS and how it maps to the web platform.
- **`spec.react.yaml` / `spec.vue.yaml` / `spec.lit.yaml` = Layer 2.** Framework-specific implementation patterns.
- **Future `spec.ios.yaml` / `spec.compose.yaml`** would require extracting a true platform-agnostic Layer 0 from current specs. This is out of scope for this workflow design.

### Vocabulary Mapping

The workflow uses the **actual spec terms**, not the idealized schema terms:

| Schema says | Actual spec uses | Workflow uses |
|-------------|------------------|---------------|
| `properties` | `attributes` | `attributes` |
| `slots` | `content` | `content` |
| `behaviors` | `interactions` | `interactions` |
| `semantics` (abstract) | `semantics` + `accessibility` | `semantics`, `accessibility` |
| (not in schema) | `visual_mapping` | `visual_mapping` |

All `review.yaml` section paths, test generation mappings, and agent prompts use the actual spec vocabulary.

### Schema Alignment (Future Work)

If cross-platform support (iOS, Android) is added later, the workflow supports a migration path:

1. Extract platform-agnostic content from `spec.yaml` into a true Layer 0
2. Create `spec.web.yaml` for web-specific mappings
3. Update the `target` field in review.yaml to support `spec-l0`, `spec-l1`, `spec-l2`

This is NOT required for the initial implementation targeting web frameworks only.

---

## Overview

### The 5-Phase Development Cycle

```
+-------------------------------------------------------+
|  Phase 1: SPEC REVIEW                                  |
|  Senior Agent reads spec -> raises questions            |
|  -> Spec gets refined before coding starts              |
+-------------------------------------------------------+
|  Phase 2: CODE GENERATION                              |
|  Coding Agent generates code from reviewed spec         |
|  -> May discover new spec issues during implementation  |
+-------------------------------------------------------+
|  Phase 3: SENIOR REVIEW                                |
|  Review Agent reads spec + code together                |
|  -> Classifies findings: "code fix" vs "spec fix"       |
+-------------------------------------------------------+
|  Phase 4: QA                                           |
|  QA Agent tests behavior against spec                   |
|  -> Classifies findings: "code bug" vs "spec gap"       |
+-------------------------------------------------------+
|  Phase 5: PARITY                                       |
|  Parity runner compares DOM across frameworks            |
|  -> Runs after all frameworks pass QA individually       |
+-------------------------------------------------------+
                      loops back
```

Every phase can produce feedback that flows back to the spec. The spec is not sacred — it is a draft that gets refined through implementation. This is what makes this system different from one-shot code generation.

---

## 1. Feedback Artifact: `review.yaml`

The glue between phases is a structured review file. Each agent produces one after its phase. This is what makes the loop machine-driven, not conversational.

### Schema

```yaml
# domains/atoms/button/.reviews/r01-p1-spec-review.yaml
phase: spec-review              # spec-review | code-gen | senior-review | qa | parity
reviewer: web-senior            # agent identifier
round: 1                        # which round of the loop
timestamp: 2026-02-12T10:30:00Z
spec_file: spec.yaml
spec_revision: 1                # which revision of the spec was reviewed

findings:
  - id: F-001
    severity: critical          # critical | high | medium | low
    target: spec                # "spec" | "code" | "spec-framework"
    target_file: spec.yaml      # which file needs changing
    section: interactions.activate
    issue: "No behavior defined when both loading AND disabled are true"
    suggestion: "Add precedence rule: loading > disabled"
    resolution: ~               # pending | fixed | deferred | wontfix | reverted

  - id: F-002
    severity: high
    target: spec
    target_file: spec.yaml
    section: composition
    issue: "Spinner dependency declared but no slot or position defined for it"
    suggestion: "Add loading-indicator content area with position rule"
    resolution: ~

  - id: F-003
    severity: low
    target: spec
    target_file: spec.yaml
    section: attributes.size
    issue: "xs size has no clear use case in the existing design system"
    suggestion: "Consider removing or documenting when xs is appropriate"
    resolution: ~

# summary is auto-generated by orchestrator, NOT by agent
```

### Key Rules

- `target: spec` = the spec.yaml needs updating
- `target: spec-framework` = a framework spec (spec.react.yaml, etc.) needs updating
- `target: code` = the code needs fixing (spec is fine)
- `resolution` field tracks how each finding was handled (updated by orchestrator)
- **Agents produce only `findings` list.** The orchestrator wraps them in the review structure, auto-generates finding IDs, and computes the summary. This prevents agents from miscounting or producing duplicate IDs.
- Review files use round-phase naming: `r01-p1-spec-review.yaml`, `r01-p3-senior-review.yaml`, `r02-p1-spec-review.yaml`

### Post-Agent Validation

After each agent produces findings, the orchestrator:

1. Parses the YAML output
2. Validates against `review.schema.yaml`
3. If invalid: re-prompts the agent once with the validation error
4. If still invalid: logs the raw output and escalates to human

---

## 2. Agent Roles Per Phase

### Phase 1: Spec Review — "Is this spec implementable?"

```
Agent:   web-senior
Input:   spec.yaml + component.schema.yaml + .conventions/{framework}.md
Output:  .reviews/rNN-p1-spec-review.yaml
Checks:
  - All required schema sections present?
  - Ambiguous behavior descriptions?
  - Missing edge cases (loading+disabled, empty content, etc.)?
  - Contradictions between sections?
  - Dependencies declared but undefined?
  - Attributes with no default value?
  - Content areas missing cardinality?
```

### Phase 2: Code Generation — "Implement this spec"

```
Agent:   web-junior (guided by .conventions/react.md)
Input:   spec.yaml + spec.react.yaml + resolved review findings
         + .conventions/react.md
Output:  Component code files (Button.tsx, Button.test.tsx, index.ts)
         + optional .reviews/rNN-p2-impl-feedback.yaml
Note:    Junior can raise issues discovered DURING coding —
         things only visible when you try to implement
```

### Phase 3: Senior Code Review — "Is this code correct AND is the spec complete?"

```
Agent:   code-reviewer (separate from Phase 1 spec-reviewer)
Input:   spec.yaml + generated code + Phase 2 feedback
         + .reviews/*.yaml (all previous reviews for context)
Output:  .reviews/rNN-p3-senior-review.yaml
Checks:
  - Code matches spec (conformance)
  - Code is idiomatic for framework
  - Performance patterns (memo, useCallback, etc.)
  - BUT ALSO: spec gaps revealed by reading the code
Note:    Different agent from Phase 1 to avoid "approving own spec" bias.
         Phase 1 = web-senior, Phase 3 = code-reviewer (or be-senior).
```

### Phase 4: QA — "Does it actually work as specified?"

```
Agent:   qa-web
Input:   spec.yaml + generated code + test results
         + .reviews/*.yaml (all previous reviews)
         + .decisions/*.md (all decisions for context)
Output:  .reviews/rNN-p4-qa-report.yaml
Checks:
  - Run Tier 1 tests (deterministic) -> pass/fail
  - Run Tier 2 tests (LLM-generated) -> pass/fail
  - Apply test gap heuristics (see Section 5)
  - Classify failures as code bug vs spec gap
  - Accessibility audit
```

### Phase 5: Parity — "Do all frameworks behave the same?"

```
Runner:  .scripts/run-parity-check.ts (automated, not an agent)
Input:   All framework code (React/Vue/Lit) + spec.yaml
Output:  .reviews/rNN-p5-parity-report.yaml
Trigger: Runs only after all frameworks individually pass QA (Phase 4)
Checks:
  - DOM structure equivalence (tag, attributes, data-*, aria-*)
  - Behavioral equivalence (event firing, state transitions)
  - Normalize framework artifacts before comparison
```

---

## 3. Loop Orchestration: When to Iterate, When to Stop

### Orchestrator State Machine

```
                  +-----------+
                  |   START   |
                  +-----+-----+
                        |
                  +-----v-----+
              +-->| Phase 1   |--[critical]---> SPEC RESOLUTION --+
              |   | Spec Rev  |                                    |
              |   +-----+-----+                                    |
              |         | [pass]                                   |
              |   +-----v-----+                                    |
              |   | Phase 2   |--[spec issue]--> SPEC RESOLUTION --+
              |   | Code Gen  |                                    |
              |   +-----+-----+                                    |
              |         | [code ready]                             |
              |   +-----v-----+                                    |
              |   | Phase 3   |--[spec issue]--> SPEC RESOLUTION --+
              |   | Sr Review |--[code issue]--> CODE FIX ---------+
              |   +-----+-----+                            |       |
              |         | [pass]                           |       |
              |   +-----v-----+                            |       |
              |   | Phase 4   |--[spec gap]---> SPEC RESOLUTION --+
              |   | QA        |--[code bug]---> CODE FIX ---+
              |   +-----+-----+                             |
              |         | [pass]                            |
              |   +-----v-----+                             |
              |   | Phase 5   |--[spec gap]---> SPEC RESOLUTION
              |   | Parity    |--[code issue]--> CODE FIX
              |   +-----+-----+
              |         | [all pass]
              |   +-----v-----+
              +---| Round++   |--[round > 3]--> ESCALATE TO HUMAN
                  | Check cap |
                  +-----+-----+
                        | [converged]
                  +-----v-----+
                  |  SHIPPED  |
                  +-----------+
```

### Iteration Rules

After each phase, check findings:

| Severity | Action |
|----------|--------|
| critical > 0 | **BLOCK.** Cannot proceed. Route back: `target: spec` -> spec-resolver, `target: code` -> coding agent fixes |
| high > 0 | **PROCEED with caution.** Accumulate. Must fix before Phase 4 (QA) starts |
| medium/low | **PROCEED.** Log as backlog in `.status.yaml` |

### Convergence Criteria

Exit when ALL of these are true:

- Phase 5 (Parity) review has 0 critical, 0 high
- No `target: spec` findings remain unresolved
- All tests pass (Tier 1 + Tier 2)
- Max iterations not exceeded (3 rounds)

### The 3-Round Cap

```
Round 1:  Spec review -> Code gen -> Senior review -> QA -> Parity
Round 2:  Fix findings -> Re-review -> Re-QA -> Re-parity
Round 3:  Final fixes -> Final QA -> SHIP or ESCALATE to human
```

If after 3 rounds there are still critical issues, the system escalates to human — something fundamental is wrong that agents cannot resolve on their own.

### Error Handling

```
Error scenario                    | Recovery
----------------------------------|---------------------------------------------
Agent network timeout             | RETRY (max 2 retries, then ESCALATE)
Agent returns malformed YAML      | Re-prompt once with validation error
                                  |   If still invalid -> ESCALATE
Spec-resolver writes invalid YAML | Validate against schema, reject + re-prompt
Code generation has syntax errors  | ROLLBACK spec to last good revision, ESCALATE
Test execution crashes             | ESCALATE (do not retry blindly)
```

Errors do NOT count toward the 3-round budget. Only completed rounds count.

### Component Status Tracking

```yaml
# domains/atoms/button/.status.yaml
component: Button
round: 2
phase: parity
status: passing             # draft | in-review | failing | passing | shipped | error

unresolved:
  critical: 0
  high: 0
  medium: 2                 # logged as backlog

backlog:
  - { id: F-003, source: r01-p1-spec-review.yaml, section: attributes.size }
  - { id: F-006, source: r01-p3-senior-review.yaml, section: code.displayName }

quality_metrics:
  coverage:
    line: 87
    branch: 82
  tests:
    total: 28
    passed: 28
    failed: 0
    tier1_count: 20         # deterministic tests
    tier2_count: 8          # LLM-generated tests
  accessibility:
    axe_violations: 0
  parity:
    structural_pass_rate: 100
  performance:
    render_time_ms: { react: 12, vue: 11, lit: 15 }
    bundle_size_kb: { react: 3.2, vue: 2.8, lit: 4.1 }

last_good_revision: 3       # last spec revision that passed QA

history:
  - { round: 1, spec_changes: 3, code_fixes: 5, duration_minutes: 45 }
  - { round: 2, spec_changes: 0, code_fixes: 1, duration_minutes: 12 }
```

---

## 4. Spec Evolution: How Specs Actually Change

When an agent says `target: spec`, the spec needs to be updated — with traceability.

### Proposed Changes

Agents provide a natural language `suggestion`. The `proposed_diff` field is **optional** — the spec-resolver agent handles the actual YAML manipulation. This is more robust than requiring reviewing agents to produce exact YAML diffs.

```yaml
findings:
  - id: F-001
    severity: critical
    target: spec
    section: interactions.activate
    issue: "No behavior for loading+disabled simultaneous state"
    suggestion: "Add precedence rule: loading state takes priority over disabled"
    # proposed_diff is optional — spec-resolver figures out the change
```

### Spec Resolver Agent

A dedicated agent processes all `target: spec` findings:

```
Agent:   spec-resolver (new role)
Input:   original spec.yaml
         + all review.yaml findings where target=spec
         + component.schema.yaml (for validation)
         + .decisions/*.md (past decisions, to avoid contradictions)
Output:  updated spec.yaml + .decisions/NNN-reason.md
Role:
  - Evaluates proposed changes (are they valid? do they conflict?)
  - Applies changes to spec.yaml
  - Validates updated spec against schema
  - Writes a decision record explaining WHY the spec changed
Guard:
  - NEVER introduce platform-specific terms not already in the spec
  - NEVER remove required schema sections
  - ALWAYS validate against component.schema.yaml after writing
```

### Post-Resolution Validation

After spec-resolver writes the updated spec.yaml, the orchestrator validates it against `component.schema.yaml`. If validation fails, the change is rejected and the spec-resolver is re-prompted with the validation errors. This is a hard gate — no invalid specs pass through.

### Conflict Resolution Rules

When multiple findings propose conflicting changes to the same section:

```
Priority order:
1. Security / accessibility issues override all others
2. Cross-component dependencies override single-component preferences
3. Later phases (QA findings) override earlier phases (spec-review)
4. Code reality (impl-discovered) overrides spec-review theory

When two findings contradict:
  1. Apply priority order above
  2. Log conflict in .decisions/ with both positions
  3. If both are critical severity -> ESCALATE to human
```

### Rollback Mechanism

If a spec change causes regression (code gen produces worse results):

```yaml
# In .status.yaml
last_good_revision: 2       # last revision that passed QA
```

The orchestrator can instruct spec-resolver to revert to `last_good_revision` if the current revision fails QA after a spec change. The revert is itself tracked as a decision record.

### Decision Records

Decision records include YAML frontmatter for machine-parseability:

```markdown
---
id: 001
component: Button
spec_revision_before: 1
spec_revision_after: 2
round: 1
phase: senior-review
finding_ids: [F-001]
affected_sections: [interactions.activate, states.loading]
decision_type: behavior-clarification  # behavior-clarification | edge-case | removal | addition
---

# Decision: Loading Precedence Rule

## Context
Senior review (round 1) found no defined behavior for loading+disabled.

## Finding
F-001 from .reviews/r01-p3-senior-review.yaml

## Decision
Loading takes precedence over disabled. When loading=true, the component
is effectively disabled (no interaction) regardless of disabled prop value.

## Spec Change
interactions.activate.condition updated to document precedence.
states section updated: loading state implicitly includes disabled behavior.
```

### Spec Versioning

Every spec change increments a revision. This requires adding `$revision` and `$changelog` as optional fields in `component.schema.yaml`.

```yaml
# At top of spec.yaml
$schema: schemas/base/component.schema.yaml
$revision: 3
$changelog:
  - { rev: 3, decision: ".decisions/003-spinner-dom-order.md", round: 2 }
  - { rev: 2, decision: ".decisions/001-loading-precedence.md", round: 1 }
  - { rev: 1, reason: "initial spec" }
```

**Schema change required:** Add optional `$revision` (number) and `$changelog` (array) fields to all base schemas. Existing specs without these fields remain valid.

---

## 5. QA Deep Dive: Test Generation from Specs

### Two-Tier Test Generation

> Addresses: WS-F002 (not fully deterministic), QA-C1 (incomplete coverage)

Spec fields are a mix of structured data (machine-parseable) and natural language descriptions. Test generation is split into two tiers:

**Tier 1: Deterministic** — generated by a script from structured spec fields. Always correct, always reproducible.

**Tier 2: LLM-assisted** — generated by a QA agent from natural language descriptions. Covers behavior semantics that cannot be derived mechanically.

### Tier 1: Deterministic Test Generation

Generated by `.scripts/generate-tests-from-spec.ts` — no LLM involved.

| Spec field (structured) | Test type | Example |
|---|---|---|
| `attributes.{name}.default` | Default rendering | `expect(el).toHaveAttribute('data-variant', 'filled')` |
| `attributes.{name}.values` (enum) | Prop variations | Render with each enum value, check data attribute |
| `visual_mapping.data_attributes` | CSS hooks | Check data-* attribute presence/value per strategy |
| `interactions.{name}.condition` | Guarded interactions | Click with loading=true -> no event fired |
| `states.{name}.aria` | ARIA state mapping | loading=true -> `aria-busy="true"` |
| `content.{name}.required` | Required content | Render without content -> warning/error |
| `composition.dependencies` | Dependency rendering | loading=true -> Spinner component present in DOM |

```typescript
// Tier 1: AUTO-GENERATED by script — do not edit
// Source: domains/atoms/button/spec.yaml rev 3

// From: attributes + defaults (deterministic)
describe('[T1] default rendering', () => {
  it('renders with data-variant=filled by default')
  it('renders with data-color=primary by default')
  it('renders with data-size=md by default')
  it('does not render data-state when no state active')
  it('does not render data-full-width when full-width=false')
})

// From: visual_mapping.data_attributes (deterministic)
describe('[T1] CSS hooks', () => {
  it('data-variant value-maps to variant attribute')
  it('data-state=loading when loading=true (priority-enum)')
  it('data-state=disabled when disabled=true and loading=false')
  it('data-state absent when neither loading nor disabled')
  it('data-full-width present-when-true')
})

// From: interactions.activate.condition (deterministic)
describe('[T1] guarded interactions', () => {
  it('click fires when loading=false AND disabled=false')
  it('click does NOT fire when loading=true')
  it('click does NOT fire when disabled=true')
})

// From: states.*.aria (deterministic)
describe('[T1] ARIA states', () => {
  it('aria-busy=true when loading')
  it('aria-disabled=true when loading')
  it('aria-disabled=true when disabled')
})
```

### Tier 2: LLM-Assisted Test Generation

Generated by the QA agent from natural language fields. These tests require semantic interpretation.

| Spec field (natural language) | Test type | Why LLM needed |
|---|---|---|
| `states.{name}.visual` | Visual behavior | "Spinner displayed inside button" — need to know what DOM to assert |
| `states.{name}.behavior` | Behavioral | "Click events ignored" — need to translate to test assertion |
| `accessibility.focus` | Focus behavior | "Remains focusable when disabled" — need framework-specific assertion |
| `accessibility.keyboard` | Keyboard nav | Key mappings to behaviors — need to compose interaction sequence |

```typescript
// Tier 2: LLM-GENERATED by qa-web agent — review before accepting
// Source: domains/atoms/button/spec.yaml rev 3

// From: states.loading.visual (LLM interpreted)
describe('[T2] loading visual behavior', () => {
  it('displays Spinner component when loading=true')
  it('hides default content when loading=true')
  it('shows content again when loading returns to false')
})

// From: accessibility.keyboard (LLM interpreted)
describe('[T2] keyboard interaction', () => {
  it('Enter key triggers activate behavior')
  it('Space key triggers activate behavior')
  it('Enter does NOT trigger when loading=true')
})

// From: accessibility.focus (LLM interpreted)
describe('[T2] focus behavior', () => {
  it('is focusable via Tab')
  it('remains focusable when disabled=true')
  it('shows default focus ring')
})
```

### Test File Structure

Separate generated and custom tests to prevent overwrites:

```
domains/atoms/button/
  __tests__/
    Button.t1.test.tsx         # Tier 1: auto-generated, overwritten on each run
    Button.t2.test.tsx         # Tier 2: LLM-generated, reviewed, then frozen
    Button.custom.test.tsx     # Custom: human/agent-written edge cases
```

### Compound Component Test Strategy

For compound components (Modal, Tabs, SegmentedControl):

```
Test structure:
  describe('parent component')        — parent-level attributes, rendering
  describe('sub-component rendering') — sub-components appear in DOM
  describe('parent-child communication') — context passing, event bubbling
  describe('keyboard navigation')     — roving tabindex, arrow keys
  describe('composition constraints') — required children, accepted types
```

Sub-component tests are generated together with the parent, not separately, because their behavior depends on the parent context.

### QA Agent Test Gap Heuristics

> Addresses: QA-C3 (concrete methodology)

The QA agent follows a structured checklist to find what the spec forgot:

**1. State Combination Matrix**

For every component with 2+ boolean attributes/states, generate the full combination matrix:

```
Button: loading(T/F) x disabled(T/F) = 4 combinations
  loading=F, disabled=F  -> spec covers (default)
  loading=T, disabled=F  -> spec covers (loading state)
  loading=F, disabled=T  -> spec covers (disabled state)
  loading=T, disabled=T  -> SPEC GAP if not explicitly documented
```

Flag missing combinations as `test_gap: true`.

**2. Boundary Value Analysis**

For every enum attribute, verify:
- Each value has distinct visual/behavioral impact
- Default value is documented
- Invalid values are handled (or spec says "constrained to enum values")

For number attributes (min/max/step): test boundaries.

**3. Accessibility Completeness**

Required checks per component:
- Every interactive element has keyboard equivalent
- Every visual state has ARIA equivalent
- Every focusable element has documented focus indicator
- Screen reader announcement defined for state changes

**4. Error / Empty State Coverage**

- What happens when required content slot is empty?
- What happens when a callback prop throws an error?
- What happens when an async operation (if any) fails?

**5. Interaction Sequence Gaps**

- Rapid repeated interactions (double-click, spam Enter)
- Interaction during state transition (click during loading animation)
- Focus management after state change (focus after modal close)

---

## 6. Cross-Framework Parity (Phase 5)

### The Parity Principle

Same `spec.yaml` generates React, Vue, and Lit components. Since all three render to the web DOM, compare DOM output rather than framework internals.

```
spec.yaml  ->  Button.tsx (React)   ->  DOM output
           ->  Button.vue (Vue)     ->  DOM output   <- must match
           ->  Button.ts  (Lit)     ->  DOM output
```

### What "Match" Means

Not pixel-perfect, but structurally equivalent:

```yaml
parity_contract:
  given:
    props: { variant: outline, size: lg, disabled: true }
  then:
    tag: button
    attributes:
      data-variant: outline
      data-size: lg
      data-state: disabled
      aria-disabled: "true"
    focusable: true
    click_fires_event: false
```

### DOM Normalization Rules

Before comparison, strip framework-specific artifacts:

| Framework | Artifacts to strip |
|---|---|
| React | `data-reactroot`, `<!-- -->` Suspense comment nodes |
| Vue | `data-v-XXXXXX` scoped CSS hashes, `<!--v-if-->` comment nodes |
| Lit | `<!--?lit-->` / `<!--lit-part-->` comment nodes, empty `<!---->` markers |

For Lit components using Shadow DOM: compare the **composed DOM tree** (flattened), not the light/shadow split.

### Parity Test Runner

```
1. Render component with props X in React  -> capture DOM snapshot
2. Render component with props X in Vue    -> capture DOM snapshot
3. Render component with props X in Lit    -> capture DOM snapshot
4. Normalize all three (strip framework artifacts per table above)
5. Compare: tag, attributes, data-*, aria-*, role, tabindex
6. Report differences as parity violations
```

**Rendering strategy:** Use Playwright (headless browser) for all three frameworks. This provides consistent rendering and supports Lit's custom element lifecycle which jsdom does not fully support. Slower than jsdom but more reliable.

### Parity Report

```yaml
# .reviews/r02-p5-parity-report.yaml
phase: parity
component: Button
round: 2
test_count: 24
pass: 22
fail: 2

violations:
  - test: "disabled state aria attributes"
    react: { aria-disabled: "true" }
    vue:   { aria-disabled: "true" }
    lit:   { disabled: "" }
    severity: high
    target: code                  # Lit implementation needs fixing

  - test: "loading spinner position"
    react: "Spinner is first child"
    vue:   "Spinner is first child"
    lit:   "Spinner is last child"
    severity: medium
    target: spec                  # spec did not define spinner position
```

### Integration with Main Loop

Parity runs as Phase 5, after all frameworks individually pass Phase 4 (QA). Parity violations:
- `target: code` -> framework coding agent fixes
- `target: spec` -> spec-resolver updates spec, all frameworks regenerate

Parity violations count toward the 3-round budget. If parity cannot be achieved within the budget, escalate to human.

---

## 7. Regression Workflow

> Addresses: QA-C2 (regression strategy missing)

### When Regression Triggers

Regression is needed when:

1. A shipped component's spec changes (`$revision` incremented after shipping)
2. A shared dependency changes (tokens, utilities, types)
3. A schema itself evolves

### Regression Detection

The orchestrator checks before starting any component cycle:

```yaml
# In .status.yaml
last_good_revision: 4          # revision when QA last passed
```

If `spec.yaml.$revision > .status.yaml.last_good_revision`, a regression cycle is needed.

### Regression Cycle

A lighter version of the full cycle — skip spec-review (the spec change is intentional):

```
1. Regenerate code from updated spec (Phase 2)
2. Run existing Tier 1 + Tier 2 tests
3. If tests fail:
   a. Classify: test expectation outdated (update test) vs code regression (fix code)
   b. QA agent updates Tier 2 tests if spec semantics changed
   c. Tier 1 tests auto-regenerate from new spec
4. Senior review of changes only (delta review, not full review)
5. Parity re-check
```

### Dependency Change Impact

When a shared module changes (e.g., token scale update):

```
1. Identify affected components via manifest.yaml dependencies
2. Run regression cycle for each affected component
3. Components can be regressed in parallel (independent)
```

### Regression Report

```yaml
# .reviews/regression-r05-report.yaml
phase: regression
trigger: spec-revision-change     # or: dependency-change, schema-change
previous_revision: 4
current_revision: 5
affected_sections: [interactions.activate, states.loading]

results:
  tier1_tests: { total: 20, passed: 18, failed: 2, updated: 2 }
  tier2_tests: { total: 8, passed: 8, failed: 0 }
  parity: pass
  status: passing
```

---

## 8. Full Walkthrough: Button Component End to End

### Round 1

```
Step 1: SPEC REVIEW (Phase 1)
  web-senior reads domains/atoms/button/spec.yaml
  -> .reviews/r01-p1-spec-review.yaml
  -> findings: 2 critical, 1 high
     F-001 [critical][spec] loading+disabled precedence undefined
     F-002 [critical][spec] Spinner content position not specified
     F-003 [high][spec]     full-width behavior on inline containers unclear
  -> blocks_next_phase: true

Step 2: SPEC RESOLUTION
  spec-resolver reads findings, updates spec.yaml
  -> spec.yaml $revision: 1 -> 2
  -> .decisions/001-button-loading-precedence.md
  -> .decisions/002-button-spinner-position.md
  -> F-003 resolution: deferred (logged in backlog)

Step 3: CODE GENERATION (Phase 2)
  web-junior reads spec.yaml (rev 2) + spec.react.yaml
  -> generates: Button.tsx, Button.test.tsx, index.ts
  -> .reviews/r01-p2-impl-feedback.yaml
     F-004 [high][spec] Spinner replaces content? Or shown alongside?

Step 4: SPEC RESOLUTION (mid-round)
  spec-resolver updates spec.yaml -> $revision: 3
  -> states.loading.visual: "Spinner replaces default content"
  web-junior regenerates Button.tsx

Step 5: SENIOR REVIEW (Phase 3, by code-reviewer, NOT web-senior)
  code-reviewer reads spec.yaml (rev 3) + Button.tsx
  -> .reviews/r01-p3-senior-review.yaml
     F-005 [high][code]  onClick handler not memoized
     F-006 [medium][code] missing displayName
  -> no spec issues

Step 6: CODE FIX
  web-junior fixes F-005, F-006

Step 7: QA (Phase 4)
  qa-web runs Tier 1 tests (script-generated): 20 tests
  qa-web generates Tier 2 tests (LLM): 8 tests
  -> .reviews/r01-p4-qa-report.yaml
  -> 27/28 pass
     F-007 [high][code] Space key fires event when loading=true
  -> applies test gap heuristics:
     F-008 [medium][spec] no edge case defined for rapid double-click

Step 8: CODE FIX
  web-junior fixes F-007
  qa-web reruns -> 28/28 pass
  F-008 resolution: deferred (backlog)
```

### Round 2: Vue + Lit + Parity

```
Same Phase 1-4 cycle for Vue and Lit generation.
(Spec is stable at rev 3, so Phase 1 is quick — no critical findings)

Phase 5: Parity runner compares all three.
-> .reviews/r02-p5-parity-report.yaml
-> 1 violation found, target: spec (spinner DOM order)
-> spec-resolver adds explicit rule -> $revision: 4
-> all three regenerated, parity passes

Final QA re-run confirms all frameworks pass.
```

### Final State

```
domains/atoms/button/
  spec.yaml                ($revision: 4, last_good_revision: 4)
  spec.react.yaml
  spec.vue.yaml
  spec.lit.yaml
  __tests__/
    Button.t1.test.tsx     (20 Tier 1 tests)
    Button.t2.test.tsx     (8 Tier 2 tests)
  .reviews/
    r01-p1-spec-review.yaml
    r01-p2-impl-feedback.yaml
    r01-p3-senior-review.yaml
    r01-p4-qa-report.yaml
    r02-p5-parity-report.yaml
  .decisions/
    001-button-loading-precedence.md
    002-button-spinner-position.md
    003-button-spinner-dom-order.md
  .status.yaml             (status: shipped, rounds: 2)
```

The spec started as a draft and evolved through 4 revisions, with full traceability.

---

## 9. Project Integration

### Directory Changes

```
woosgem-mda/
  schemas/
    base/                          # existing (5 files) — add $revision, $changelog
    lang/                          # existing (3 files)
    platform/                      # existing (4 files)
    framework/                     # existing (3 files)
    workflow/                      # NEW
      review.schema.yaml           #   review finding format
      status.schema.yaml           #   component lifecycle + quality metrics
      parity.schema.yaml           #   cross-framework parity report
      decision.schema.yaml         #   decision record frontmatter

  domains/atoms/button/
    spec.yaml                      # existing (add $revision, $changelog)
    spec.react.yaml                # existing
    spec.vue.yaml                  # existing
    spec.lit.yaml                  # existing
    __tests__/                     # NEW (per-component)
      Button.t1.test.tsx           #   Tier 1: deterministic (auto-overwritten)
      Button.t2.test.tsx           #   Tier 2: LLM-generated (frozen after review)
      Button.custom.test.tsx       #   Custom edge cases
    .reviews/                      # NEW (per-component, git-tracked)
    .decisions/                    # NEW (per-component, git-tracked)
    .status.yaml                   # NEW

  .scripts/
    run-component-cycle.ts         # NEW — orchestrator state machine
    generate-tests-from-spec.ts    # NEW — Tier 1 test generator
    run-parity-check.ts            # NEW — parity runner (Playwright)
    validate-review.ts             # NEW — review.yaml schema validation

  .conventions/                    # NEW (must exist before first cycle)
    react.md                       #   React code gen rules
    vue.md                         #   Vue code gen rules
    lit.md                         #   Lit code gen rules
```

### New Agent Configs

```
configs/
  spec-resolver.md                 # NEW — resolves spec findings
  code-reviewer.md                 # NEW — Phase 3 (distinct from spec-reviewer)
  # existing agents get workflow-aware prompts:
  web-senior.md                    # add: review.yaml output format, Phase 1 role
  web-junior.md                    # add: reads review findings, raises impl feedback
  qa-web.md                        # add: two-tier test generation, gap heuristics
```

### manifest.yaml Addition

```yaml
workflow:
  schemas:
    review: schemas/workflow/review.schema.yaml
    status: schemas/workflow/status.schema.yaml
    parity: schemas/workflow/parity.schema.yaml
    decision: schemas/workflow/decision.schema.yaml
  scripts:
    orchestrator: .scripts/run-component-cycle.ts
    test_generator: .scripts/generate-tests-from-spec.ts
    parity_runner: .scripts/run-parity-check.ts
    review_validator: .scripts/validate-review.ts
  agents:
    spec_reviewer: web-senior       # Phase 1
    code_generator: web-junior      # Phase 2
    senior_reviewer: code-reviewer  # Phase 3 (distinct from Phase 1)
    qa: qa-web                      # Phase 4
    spec_resolver: spec-resolver    # Spec evolution
  rules:
    max_rounds: 3
    block_on: critical
    escalate_to: human
    supervised_mode: true           # human approves spec changes (first 3 components)
  limitations:
    - "Frameworks processed sequentially for spec evolution (not parallel)"
    - "Parallel code gen only for stable specs (no pending target:spec findings)"
```

---

## 10. Implementation Roadmap

### Phase 0: Prerequisites (before anything else)

0. **`.conventions/` skeleton files** — `react.md`, `vue.md`, `lit.md`
   - Minimum: file structure, import conventions, factory pattern reference
   - Without these, code generation agents have no style guide

### Phase A: Foundation (enables the feedback loop)

1. **Workflow schemas** — `schemas/workflow/review.schema.yaml`, `status.schema.yaml`, `parity.schema.yaml`, `decision.schema.yaml`
   - Defines the structure all agents must follow
   - Severity levels, target classification, resolution tracking

2. **Schema update** — Add optional `$revision` and `$changelog` to base schemas
   - Non-breaking: existing specs without these fields remain valid

3. **Agent configs** — `spec-resolver.md`, `code-reviewer.md`
   - spec-resolver: conflict resolution rules, schema validation guard
   - code-reviewer: separate from spec-reviewer to avoid bias

4. **Review validator** — `.scripts/validate-review.ts`
   - Validates agent output against review.schema.yaml
   - Used by orchestrator after every agent call

### Phase B: The Loop Engine

5. **Orchestrator script** — `.scripts/run-component-cycle.ts`
   - State machine implementation (see Section 3 diagram)
   - Agent invocation (via subprocess calls to `claude` CLI)
   - YAML parsing (`js-yaml`), review validation, convergence check
   - Error handling with retry/escalate logic
   - `.status.yaml` management

### Phase C: Test Generation

6. **Tier 1 test generator** — `.scripts/generate-tests-from-spec.ts`
   - Deterministic script, no LLM
   - Reads spec.yaml structured fields -> generates vitest assertions
   - Covers: defaults, enum variations, CSS hooks, ARIA states, guarded interactions

7. **QA agent heuristics** — Update `qa-web.md` config
   - State combination matrix generation
   - Boundary value analysis
   - Accessibility completeness checklist
   - Tier 2 test generation prompt

### Phase D: Parity

8. **Parity test runner** — `.scripts/run-parity-check.ts`
   - Playwright-based rendering for all three frameworks
   - DOM normalization (per framework artifact stripping table)
   - Structural comparison and parity report generation

### Suggested Timeline

| Week | Phase | Deliverable |
|------|-------|-------------|
| 1 | 0 + A | .conventions/ skeletons, workflow schemas, agent configs, review validator |
| 2 | B | Orchestrator state machine |
| 3 | C | Tier 1 test generator + QA agent heuristics |
| 4-5 | D | Parity runner (Playwright setup + normalization) |

**Milestone:** Run full cycle on Button component end-to-end.

**Supervised mode:** First 3 components (Button, Icon, Spinner) run with human approval on spec-resolver changes. After calibration, switch to fully autonomous mode.

### Component Processing Order

```
Phase 1: Atoms (11 components) — independent, can run in parallel
Phase 2: Molecules (9 components) — depend on atoms, wait for atom dependencies to ship
Phase 3: Organisms (5 components) — depend on atoms + molecules
```

---

## 11. Risks and Mitigations

### Risk 1: Infinite Feedback Loops

Agent A says "fix spec." Spec-resolver fixes it. Agent B says "that change broke something else."

**Mitigation:** Hard cap at 3 rounds. Decision records prevent flip-flopping (spec-resolver reads past decisions). Round 3 criticals escalate to human. Rollback to `last_good_revision` if a spec change makes things worse.

### Risk 2: Review Inflation

Agents over-report to seem thorough. 50 medium findings that are mostly stylistic noise.

**Mitigation:** Only critical/high block progress. Medium/low go to backlog with explicit tracking in `.status.yaml`. Agent prompts state: "Only report issues that would cause bugs or user-facing problems. Style preferences are not findings."

### Risk 3: Spec Over-Specification

Every edge case gets added through feedback. spec.yaml grows from 100 to 500+ lines.

**Mitigation:** Spec covers WHAT and WHY, never HOW. Implementation details stay in `.conventions/`. If a section is only relevant to one framework, it belongs in `spec.react.yaml`, not `spec.yaml`. Periodic spec hygiene pass.

### Risk 4: Test Generation Quality

Tier 1 tests only cover structured fields. Tier 2 tests depend on LLM quality.

**Mitigation:** Tier 1 provides a reliable baseline. Tier 2 tests are reviewed and frozen (not regenerated every run). QA agent heuristics systematically find gaps. Human QA review on first 3 components to calibrate.

### Risk 5: Cost (Many Agent Calls)

5 phases x 3 frameworks x up to 3 rounds = many LLM calls for 25 components.

**Mitigation:**
- Start with 1 component (Button) as pilot
- Atoms first, then molecules, then organisms
- Cache: if spec unchanged between rounds, skip spec-review
- Parallelize: React/Vue/Lit code gen runs concurrently (when spec is stable)
- Track agent invocation count per component in `.status.yaml`
- Soft budget: flag components exceeding 20 agent calls for human review

**Rough cost estimate:** ~5K-8K input tokens per agent call, ~1K-2K output tokens. Worst case (all 25 components, max rounds): ~900 calls. Realistic with caching: ~300-400 calls.

### Risk 6: Agent YAML Output Reliability

LLMs may produce malformed YAML or schema-drifted output.

**Mitigation:** Orchestrator validates every agent output against schema. Re-prompt once on failure. Auto-generate summary/IDs in orchestrator (agents only produce findings list). Consider JSON output from agents + post-hoc YAML conversion if YAML errors persist.

### The Honest Trade-Off

| | Without workflow | With workflow |
|---|---|---|
| Speed | Fast generation | Slower per-component |
| Quality | Unknown, manual debugging later | Verified, documented, consistent |
| Traceability | None | Full audit trail (reviews + decisions) |
| Break-even | — | ~5 components |

The upfront cost pays for itself because you stop finding cross-framework inconsistencies weeks later.

---

## Appendix A: Senior Review Summary

This document was reviewed by 3 senior agents. Key changes from v1 to v2:

| Issue | Raised By | Resolution |
|---|---|---|
| Spec terminology mismatch | BE, Web, QA | Added "Layer Model Reality" section — accept merge, use actual terms |
| Missing Layer 1 (spec.web.yaml) | BE, Web | Documented as intentional merge for web-only scope |
| Test generation not fully deterministic | Web | Split into Tier 1 (deterministic) + Tier 2 (LLM-assisted) |
| Regression strategy missing | QA | Added Section 7: Regression Workflow |
| QA heuristics undefined | QA | Added concrete test gap detection checklist |
| Orchestrator error handling | BE, Web | Added error handling table + state machine diagram |
| Parity not in main loop | BE | Promoted to Phase 5 (was separate) |
| Dual-role agent bias | BE | Split: web-senior (Phase 1) vs code-reviewer (Phase 3) |
| proposed_diff fragility | BE, Web | Made optional; spec-resolver handles YAML manipulation |
| No rollback mechanism | BE | Added `last_good_revision` + revert capability |
| $revision not in schema | Web | Noted as required schema change in Phase A |
| review.yaml validation | Web | Added post-agent validation step + orchestrator auto-generates IDs |
| Spec-resolver conflicts | QA | Added priority-based conflict resolution rules |
| .conventions/ missing | BE, Web | Added as Phase 0 prerequisite |
| QA metrics insufficient | QA | Expanded .status.yaml with quality_metrics section |
| Decision record format | QA | Added YAML frontmatter for machine-parseability |
| Review file naming | BE | Changed to round-phase encoding: `r01-p1-spec-review.yaml` |
| Supervised mode | BE | Added for first 3 components — human approves spec changes |
| Visual regression testing | QA | Noted as future enhancement (post Phase D) |
| Performance benchmarking | QA | Added to quality_metrics in .status.yaml |
