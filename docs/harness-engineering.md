# Harness Engineering for woosgem-mda

> Designing the environment where agents do reliable work

**Status:** concept
**Date:** 2026-02-12
**Inspired by:** OpenAI's Harness Engineering — "leveraging Codex in an agent-first world"

---

## 0. Why This Document Exists

Three documents govern how agents work in this project. Each answers a different question:

| Document | Question | Audience |
|----------|----------|----------|
| **This document** | WHY is the project structured this way? | Architects, orchestrator designers |
| [Workflow Design](plans/2026-02-12-spec-driven-dev-workflow-design.md) | HOW do agents execute the 5-phase cycle? | Agent implementers, orchestrator script |
| `CLAUDE.md` | WHAT does an agent need to know to start working? | Every agent at session start |

The Workflow Design document (1,189 lines) defines phases, review artifacts, iteration rules, and convergence criteria. It assumes an environment that makes agent work reliable. This document designs that environment.

**Central thesis:** The manifest-driven architecture already provides an unusually strong foundation for agent legibility. The `manifest.yaml` -> domain manifest -> component spec navigation chain is a near-ideal "map" structure. The harness work is about filling specific gaps: context budgeting, tiered information loading, handoff artifacts, and missing infrastructure files.

---

## 1. The Harness Philosophy Applied to woosgem-mda

### 1.1 What is a Harness?

Not a generic definition. In this project specifically, the harness is the set of files, conventions, and navigation structures that allow any agent to:

1. **Find** the right spec for a component in under 3 file reads
2. **Know** what schema governs that spec without external documentation
3. **Understand** its role in the workflow without reading the full workflow design
4. **Produce** structured output that the next agent (or orchestrator) can parse
5. **Operate** within a context budget that fits the task

The harness is not a framework or library. It is the repository structure itself — optimized for agent consumption.

### 1.2 "Map, Not Manual"

The first principle: give agents a navigation structure, not a comprehensive manual.

**The woosgem-mda map:**

| File | Role | Lines |
|------|------|-------|
| `manifest.yaml` | Project-level map: domains, schemas, shared modules | 93 |
| `domains/atoms/manifest.yaml` | Domain map: 11 components, specs, contracts | 74 |
| `domains/molecules/manifest.yaml` | Domain map: 9 components | 65 |
| `domains/organisms/manifest.yaml` | Domain map: 5 components | 57 |
| `shared/manifest.yaml` | Shared module map: tokens, CSP, utils, types | 132 |
| **Total navigation** | **Full project structure** | **421** |

An agent can understand the entire project layout by reading 421 lines of YAML. This is the map. The component specs (49-266 lines each), schemas (215-478 lines each), and shared specs (232-522 lines each) are the territory — visited only when needed.

### 1.3 "Anything Not In-Context Doesn't Exist"

The second principle: if an agent cannot access something in its current context window, it effectively does not exist. This has concrete implications:

- An agent doing **spec review** for Button needs: `button/spec.yaml` (104 lines) + `component.schema.yaml` (391 lines) = **495 lines**. It does NOT need the web schema, React schema, or any other component's spec.
- An agent doing **React code gen** for Button needs: spec (104) + web schema (243) + react schema (459) + TS schema (215) + conventions (~120) = **~1,141 lines**. This is the full context for one agent, one task, one component.
- An agent doing **QA** needs spec + code + tests + previous reviews. This can exceed 2,000 lines, which is why tiered loading matters.

The takeaway: every piece of context loaded must justify its presence. Loading "just in case" wastes the agent's most scarce resource.

### 1.4 Depth-First Decomposition

The third principle: break goals into building blocks that agents construct sequentially.

```
Goal: Ship Button component for React
  -> Block 1: Review Button spec         (spec.yaml only)
  -> Block 2: Generate React code        (spec + react schema + conventions)
  -> Block 3: Review generated code      (spec + code + reviews)
  -> Block 4: Run QA                     (spec + code + tests + reviews)
  -> Block 5: Parity check               (all frameworks + spec)
```

Each block is a self-contained agent task with a defined input set and a structured output (`review.yaml` or code files). No block requires the full project context. This maps directly to the 5-phase workflow.

---

## 2. Context Optimization (Pillar 1)

### 2.1 Context Budget Analysis

Measured from the actual codebase:

**Component spec sizes (Layer 0 — spec.yaml):**

| Category | Range | Average | Examples |
|----------|-------|---------|---------|
| Simple atoms | 49-65 lines | ~57 | Kbd (49), Spinner (50), Icon (57) |
| Medium atoms | 74-104 lines | ~87 | Avatar (74), Progress (87), Button (104) |
| Molecules | 95-266 lines | ~145 | Alert (95), Input (122), Select (266) |
| Organisms | 146-183 lines | ~165 | SegmentedControl (146), Tab (183) |

**Schema sizes:**

| Schema | Lines | When needed |
|--------|-------|-------------|
| `base/component.schema.yaml` | 391 | Spec review (Phase 1) |
| `platform/web.schema.yaml` | 243 | Code gen (Phase 2+) |
| `framework/react.schema.yaml` | 459 | React code gen |
| `framework/vue.schema.yaml` | 413 | Vue code gen |
| `framework/lit.schema.yaml` | 478 | Lit code gen |
| `lang/typescript.schema.yaml` | 215 | Any TS code gen |

**Shared spec sizes:**

| Module | Lines | When needed |
|--------|-------|-------------|
| `tokens/spec.yaml` | 438 | Token reference (rarely in full) |
| `csp/spec.yaml` | 522 | Color work only |
| `utils/spec.yaml` | 232 | Utility reference |
| `types/spec.yaml` | 304 | Type reference |

### 2.2 Context Budget Per Agent Task

Explicit budgets by phase:

| Agent Task | Required Context | Budget (lines) |
|------------|-----------------|----------------|
| **Spec Review** (Phase 1) | spec + component schema | 450-650 |
| **React Code Gen** (Phase 2) | spec + web/react/TS schemas + conventions | 1,100-1,300 |
| **Senior Review** (Phase 3) | spec + generated code + reviews | 500-1,000 |
| **QA** (Phase 4) | spec + code + tests + reviews | 1,000-2,500 |
| **Parity** (Phase 5) | automated script, not agent | N/A |

Phase 2 (code gen) is the heaviest single-agent load. Phase 4 (QA) can exceed it when reviews accumulate across rounds.

### 2.3 Tiered Information Loading

**The problem:** An agent doing React code gen loads the full `component.schema.yaml` (391 lines) even though it only needs structural awareness, not every field definition.

**The solution: Three loading tiers.**

**Tier 1: Briefing** (always loaded, ~20 lines)
A compact per-component summary. The "agent's clipboard" — what it needs to start working immediately.

```yaml
# domains/atoms/button/.briefing.yaml (auto-generated, do not edit)
component: Button
category: atom
complexity: simple
spec: spec.yaml                    # 104 lines
schema: schemas/base/component.schema.yaml
dependencies: [Spinner]
domain: atoms
context_hints:
  spec_review: [spec.yaml, schemas/base/component.schema.yaml]
  react_codegen: [spec.yaml, schemas/platform/web.schema.yaml,
                  schemas/framework/react.schema.yaml, .conventions/react.md]
  qa: [spec.yaml, __tests__/*.test.tsx, .reviews/*.yaml]
```

**Tier 2: Essential** (loaded on demand)
The actual spec file + the schema relevant to the current task.

**Tier 3: Reference** (loaded only when stuck)
Full schemas, other component specs for cross-referencing, shared module specs.

**Loading sequence:**

```
1. Read .briefing.yaml                    (Tier 1, ~20 lines)
2. Read spec.yaml                         (Tier 2, 50-266 lines)
3. Read task-specific schema summary      (Tier 1, ~40 lines)
4. Read .conventions/{framework}.md       (Tier 2, ~100-150 lines)
5. Read previous review findings if any   (Tier 2, variable)
6. [Only if stuck] Load full schema       (Tier 3, 215-478 lines)
```

Initial load without reviews: ~210-476 lines. Intentionally small. Agents can always load more, but the default is lean.

### 2.4 Schema Summaries

Compact schema summaries (~40 lines) alongside full schemas (215-478 lines). Agents load summaries for structural awareness; full schemas only for validation.

```yaml
# schemas/base/component.schema.summary.yaml (~40 lines vs 391)
required_sections: [component, attributes, content, states, interactions, semantics]
optional_sections: [composition, control]
attribute_types: [enum, boolean, string, number, union, callback]
interaction_effects: [emit, toggle, set]
semantic_roles: [button, checkbox, radio, switch, textbox, listbox, dialog, ...]
content_types: [text, node, component-list]
data_attribute_strategies: [value-mapped, priority-enum, present-when-true, ...]
```

This reduces schema context from 391 lines to ~40 lines for agents that need "what sections exist" but not "every field definition."

### 2.5 Agent Briefing Files

Two types:

**Per-component briefing** (`.briefing.yaml`): auto-generated from spec + manifest. Located at `domains/{category}/{component}/.briefing.yaml`. Tells any agent what files to load for a specific component.

**Per-role briefing** (in `.conventions/` or agent config): tells a specific agent type what it needs regardless of component. Maps to existing agent config system.

### 2.6 The .conventions/ Directory

A critical gap. Referenced by both the workflow design and CLAUDE.md but does not exist yet. This is the single most important missing piece — without it, no code generation agent can produce consistent output.

Proposed structure:

```
.conventions/
  react.md          # React code gen rules (~100-150 lines)
  vue.md            # Vue code gen rules
  lit.md            # Lit code gen rules
  shared.md         # Common patterns across frameworks
  naming.md         # CSS class naming, file naming, export conventions
```

Each file is a compact set of rules, not a tutorial. An agent reads `react.md` and knows: file structure, props pattern, ref pattern, state management, export pattern, testing conventions.

### 2.7 What NOT to Optimize

Not everything needs compression. These work well and should stay as-is:

- **manifest.yaml chain** (421 lines): excellent project navigation. Do not compress.
- **$schema declarations**: every spec self-describes its governing schema. Best possible discoverability.
- **Schema files themselves**: their verbosity serves validation. The optimization is adding summaries alongside, not shrinking them.

---

## 3. Multi-Agent Orchestration (Pillar 2)

### 3.1 Agent Roster

Five roles mapped to the 5-phase workflow:

| Agent | Phase | Config Status | Key Context |
|-------|-------|---------------|-------------|
| `web-senior` | 1: Spec Review | EXISTS | spec + component schema |
| `web-junior` | 2: Code Gen | EXISTS | spec + framework schemas + conventions |
| `code-reviewer` | 3: Senior Review | MISSING | spec + code + reviews |
| `qa-web` | 4: QA | EXISTS | spec + code + tests + reviews |
| `spec-resolver` | Spec Evolution | MISSING | spec + findings + decisions + schema |

Two new agent configs needed: `code-reviewer` (distinct from spec-reviewer to avoid bias) and `spec-resolver` (processes spec findings, writes decision records).

### 3.2 Handoff Artifact: review.yaml

The glue between phases. The harness concern is how agents reliably find, read, and write these files.

**Location convention:** `domains/{category}/{component}/.reviews/rNN-pN-{phase}.yaml`

**Discovery:** the orchestrator tells each agent exactly which review files to read. Agents never search for reviews.

**Validation:** the orchestrator validates every review output against `schemas/workflow/review.schema.yaml` before passing it to the next agent. Invalid output triggers one re-prompt, then escalation.

**Agent output scope:** agents produce only the `findings` list. The orchestrator wraps them in review structure, auto-generates finding IDs, and computes summaries.

### 3.3 Agent Context Loading Protocol

Every agent follows this protocol at session start:

```
Step 1: Receive handoff payload from orchestrator
        (component briefing + relevant files + previous findings summary)

Step 2: Read component spec.yaml
        (the source of truth for what this component IS)

Step 3: Read task-specific context
        - Phase 1: component schema (or summary)
        - Phase 2: framework schemas + .conventions/{fw}.md
        - Phase 3: generated code + Phase 1-2 reviews
        - Phase 4: code + tests + all reviews + decisions

Step 4: Produce structured output
        - findings list (YAML) for review phases
        - code files for code gen phase
        - test files for QA phase

Step 5: Return output to orchestrator for validation
```

The agent never decides what to load. The orchestrator decides based on the phase and component.

### 3.4 Orchestrator Responsibilities

The orchestrator (`.scripts/run-component-cycle.ts`) is the harness's central controller:

1. **Pre-computes the file list** each agent needs (from `.briefing.yaml` context hints)
2. **Assembles the handoff payload** — file contents, not file paths
3. **Validates outputs** against workflow schemas
4. **Manages .status.yaml** — agents never write to this
5. **Auto-generates review metadata** — IDs, timestamps, summaries
6. **Enforces gating logic** — critical findings block phase progression
7. **Tracks convergence** — round count, spec revision, resolution status

### 3.5 Handoff Payload Structure

What the orchestrator passes between phases:

```yaml
handoff:
  component: Button
  phase: senior-review
  round: 1
  spec_revision: 3
  files:
    spec: domains/atoms/button/spec.yaml
    code: domains/atoms/button/internals/react/Button.tsx
    reviews:
      - .reviews/r01-p1-spec-review.yaml
      - .reviews/r01-p2-impl-feedback.yaml
  previous_findings_summary:
    critical: 0
    high: 1
    spec_changes_applied: 2
  instructions: .conventions/react.md
```

The orchestrator constructs this. The agent receives it as initial context. No filesystem navigation required.

### 3.6 The "No Search" Principle

A key harness design decision: **agents should never need to search the filesystem for context.** The orchestrator provides everything.

This eliminates:
- Agents loading wrong files
- Agents loading too many files
- Agents wasting context on file discovery operations
- Non-deterministic agent behavior based on search results

This is possible because the manifest system makes all file locations deterministic. Given a component name and a phase, the exact set of required files is computable.

---

## 4. Relationship to Existing Documents

### 4.1 Document Hierarchy

```
CLAUDE.md (agent entry point)
  |
  +-- manifest.yaml (navigation map)
  |     +-- domain manifests (what exists)
  |     +-- schema index (how specs are structured)
  |
  +-- docs/harness-engineering.md (THIS DOCUMENT)
  |     +-- WHY the project is structured this way
  |     +-- Context optimization principles
  |     +-- Multi-agent orchestration model
  |
  +-- docs/plans/spec-driven-dev-workflow-design.md
        +-- HOW agents execute the 5-phase cycle
        +-- review.yaml schema and lifecycle
        +-- QA test generation strategy
        +-- Parity checking approach
```

### 4.2 What This Adds to the Workflow Design

The harness document does NOT replace the workflow design. It adds a layer beneath it:

| Gap in Workflow Design | Harness Addition |
|------------------------|------------------|
| "Agent reads spec.yaml" — but which schemas to load? | Context hints in `.briefing.yaml` |
| Assumes agents load full schemas | Introduces compact schema summaries |
| Describes what agents do | Adds what the orchestrator prepares before invocation |
| References `.conventions/react.md` | Defines .conventions/ structure and content guidelines |
| Defines review.yaml structure | Adds handoff payload wrapping and "No Search" principle |

---

## 5. Implementation Priority

### Priority 1: Foundation (enables the feedback loop)

- **`.conventions/react.md`** — without this, no code gen agent can produce consistent output
- **`schemas/workflow/review.schema.yaml`** — without this, no agent output can be validated
- **`spec-resolver` agent config** — without this, spec evolution cannot happen

### Priority 2: Context Optimization

- **Schema summaries** (`*.schema.summary.yaml`) — reduces code gen context by 400-600 lines
- **`.briefing.yaml` generator** (`.scripts/generate-briefings.ts`) — auto-generates per-component briefings
- **`.conventions/vue.md` + `.conventions/lit.md`** — enables multi-framework code gen

### Priority 3: Orchestration Infrastructure

- **`schemas/workflow/status.schema.yaml`** — component lifecycle tracking
- **Orchestrator skeleton** (`.scripts/run-component-cycle.ts`) — state machine shell
- **Agent config updates** — add workflow awareness to web-senior, web-junior, qa-web

### Priority 4: Workflow Activation

- **`code-reviewer` agent config** — enables Phase 3
- **Tier 1 test generator** (`.scripts/generate-tests-from-spec.ts`)
- **Button pilot** — end-to-end run on one component

### Dependency Graph

```
.conventions/react.md ──────────────┐
review.schema.yaml ─────────────────┤
spec-resolver config ───────────────┤
                                    v
                    Pilot: Button spec review (Phase 1)
                                    |
schema summaries ───────────────────┤
.briefing.yaml generator ──────────┤
                                    v
                    Pilot: Button code gen (Phase 2)
                                    |
status.schema.yaml ────────────────┤
code-reviewer config ──────────────┤
                                    v
                    Pilot: Button review + QA (Phase 3-4)
                                    |
.conventions/vue.md ───────────────┤
.conventions/lit.md ───────────────┤
Tier 1 test generator ────────────┤
                                    v
                    Multi-framework + Parity (Phase 5)
```

---

## 6. Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Agent context load (code gen) | < 1,500 lines | Sum lines in handoff payload |
| Agent file discovery operations | 0 | Agents receive contents, never search |
| Review validation pass rate | > 90% first attempt | Orchestrator valid/invalid ratio |
| Average rounds per component | <= 2 | `.status.yaml` history |
| Average spec revisions per component | <= 4 | `spec.yaml $revision` |

---

## Appendix A: Context Budget Cheat Sheet

```
Navigation (always small):
  manifest.yaml              93 lines
  domain manifests           57-74 lines each
  .briefing.yaml             ~20 lines

Component specs:
  Simple atom                ~55 lines
  Complex atom               ~100 lines
  Molecule                   ~145 lines
  Organism                   ~165 lines

Schemas (full):
  component.schema           391 lines
  web.schema                 243 lines
  react.schema               459 lines
  vue.schema                 413 lines
  lit.schema                 478 lines
  typescript.schema          215 lines

Schemas (summary):
  Any summary                ~40 lines

Shared specs:
  tokens                     438 lines
  csp                        522 lines
  utils                      232 lines
  types                      304 lines

Worst case (React code gen, full schemas):  ~1,300 lines
Typical agent task:                         ~500-800 lines
Best case (spec review with summary):       ~170 lines
```

## Appendix B: Glossary

| Term | Meaning in woosgem-mda |
|------|----------------------|
| **Harness** | The infrastructure (files, scripts, conventions) that enables agents to work reliably |
| **Briefing** | Auto-generated compact summary of what an agent needs for a specific component |
| **Context budget** | Maximum lines of source material an agent should load for a task |
| **Handoff payload** | File contents + metadata the orchestrator passes between agents |
| **Tier 1/2/3** | Information loading tiers: briefing / essential / reference |
| **Map files** | The manifest.yaml chain — navigational structure |
| **Territory files** | spec.yaml, schema.yaml — actual content agents work with |
| **No Search** | Principle that agents receive all needed files from orchestrator, never search |
