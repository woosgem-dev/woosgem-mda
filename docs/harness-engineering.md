# Harness Engineering

> Designing the environment where agents do reliable work.
> The manifest gives direction. The harness makes execution predictable.

A companion to the [MDA Playbook](./playbook.md).
The Playbook defines **how to structure** an MDA project.
This document defines **how to optimize that structure** for agent execution.

---

## Table of Contents

1. [What is a Harness?](#1-what-is-a-harness)
2. [Core Principles](#2-core-principles)
3. [Context Optimization](#3-context-optimization)
4. [Handoff Artifacts](#4-handoff-artifacts)
5. [Orchestration](#5-orchestration)
6. [Conventions Directory](#6-conventions-directory)
7. [Applying to Your Project](#7-applying-to-your-project)
8. [Success Metrics](#8-success-metrics)
9. [Glossary](#9-glossary)

---

## 1. What is a Harness?

The harness is the set of files, conventions, and navigation structures
that allow any agent to:

1. **Find** the right spec in under 3 file reads
2. **Know** what schema governs that spec without external documentation
3. **Understand** its role in the workflow without reading the full workflow design
4. **Produce** structured output that the next agent (or orchestrator) can parse
5. **Operate** within a context budget that fits the task

The harness is not a framework or library.
It is the repository structure itself — optimized for agent consumption.

### Relationship to the Playbook

| Document | Question |
|----------|----------|
| **Playbook** | HOW is the project structured? |
| **Harness Engineering** | WHY is it structured that way, and how do agents work efficiently within it? |

The Playbook designs the map. This document designs the operating environment.

---

## 2. Core Principles

### 2.1 Map, Not Manual

Give agents a navigation structure, not a comprehensive manual.

The manifest chain IS the map:

```
manifest.yaml                  (project-level: domains, config paths)
  -> specs/{domain}/manifest.yaml   (domain-level: modules, rules)
    -> {module}/spec.yaml           (module-level: actual content)
```

An agent can understand the entire project layout by reading
only the manifest chain. This is intentionally small.
The spec files, schemas, and shared modules are the territory —
visited only when needed.

**Measure your map.** Count the total lines of all manifest files combined.
If an agent can understand your project structure in under 500 lines of YAML,
your map is working.

### 2.2 Anything Not In-Context Doesn't Exist

If an agent cannot access something in its current context window,
it effectively does not exist.

Implications:

- Every piece of context loaded must justify its presence
- Loading "just in case" wastes the agent's most scarce resource
- An agent doing spec review does NOT need code generation schemas
- An agent generating code does NOT need other modules' specs

**The question to always ask:** Does this agent need this file for THIS task?
If no, don't load it.

### 2.3 Depth-First Decomposition

Break goals into building blocks that agents construct sequentially.
Each block is a self-contained task with a defined input set
and a structured output.

```
Goal: Ship feature X
  -> Block 1: Review spec           (spec only)
  -> Block 2: Generate code         (spec + schemas + conventions)
  -> Block 3: Review generated code (spec + code + reviews)
  -> Block 4: Run QA                (spec + code + tests + reviews)
```

No block requires the full project context.
Each block maps to a phase in the MDA development cycle.

### 2.4 No Search

**Agents should never need to search the filesystem for context.**

The manifest system makes all file locations deterministic.
Given a module name and a phase, the exact set of required files
is computable. The orchestrator (or the human) provides everything.

This eliminates:
- Agents loading wrong files
- Agents loading too many files
- Agents wasting context on file discovery operations
- Non-deterministic agent behavior based on search results

---

## 3. Context Optimization

### 3.1 Context Budget

Every agent task has a context budget — the maximum lines of source material
it should load. Exceeding the budget degrades agent performance.

**Budget guidelines by MDA cycle phase:**

| Phase | Required Context | Typical Budget |
|-------|-----------------|----------------|
| Spec Review | spec + governing schema | Small |
| Code Generation | spec + target schemas + conventions | Large (heaviest) |
| Senior Review | spec + generated code + reviews | Medium |
| QA | spec + code + tests + reviews | Large (grows with rounds) |

To establish budgets for your project:
1. Measure the line count of each spec, schema, and convention file
2. Sum the files needed per phase
3. If any phase exceeds ~1,500 lines, introduce tiered loading

### 3.2 Tiered Information Loading

Three loading tiers to control what agents see and when.

**Tier 1: Briefing** (always loaded first)

A compact per-module summary. The "agent's clipboard."
What it needs to start working immediately.

```yaml
# specs/{domain}/{module}/.briefing.yaml (auto-generated)

module: order-cancel
domain: order
spec: spec.yaml
schema: schemas/business-logic.schema.yaml
dependencies: [order-types, product-stock]
context_hints:
  spec_review: [spec.yaml, schemas/business-logic.schema.yaml]
  code_gen: [spec.yaml, schemas/typescript.schema.yaml, conventions/api.yaml]
  qa: [spec.yaml, __tests__/*.test.ts, .reviews/*.yaml]
```

**Tier 2: Essential** (loaded on demand)

The actual spec file + the schema relevant to the current task.

**Tier 3: Reference** (loaded only when stuck)

Full schemas, other module specs for cross-referencing, shared specs.

**Loading sequence:**

```
1. Read .briefing.yaml                    (Tier 1)
2. Read spec.yaml                         (Tier 2)
3. Read task-specific schema summary      (Tier 1)
4. Read conventions for current task      (Tier 2)
5. Read previous review findings if any   (Tier 2)
6. [Only if stuck] Load full schema       (Tier 3)
```

The default is lean. Agents can always load more, but start small.

### 3.3 Schema Summaries

When schemas are large (200+ lines), create compact summaries (~40 lines)
alongside the full schemas. Agents load summaries for structural awareness;
full schemas only for validation.

```yaml
# schemas/component.schema.summary.yaml

required_sections: [component, attributes, states, interactions]
optional_sections: [composition, control]
attribute_types: [enum, boolean, string, number, callback]
# ... compact overview of what the schema expects
```

This can reduce schema context by 80% for agents that need
"what sections exist" but not "every field definition."

### 3.4 What NOT to Optimize

Not everything needs compression:

- **Manifest chain** — excellent navigation. Do not compress.
- **$schema declarations** — self-describing specs are valuable. Keep them.
- **Schema files themselves** — their verbosity serves validation. Add summaries alongside, don't shrink them.

---

## 4. Handoff Artifacts

### 4.1 Purpose

Handoff artifacts are structured files that carry context
between phases of the MDA cycle.
They are the glue between agents working on the same module.

### 4.2 Review Files

The primary handoff artifact. Written by review agents, consumed by the next phase.

**Location convention:**

```
specs/{domain}/{module}/.reviews/rNN-pN-{phase}.yaml
```

- `rNN` — round number (r01, r02, ...)
- `pN` — phase number (p1 = spec review, p2 = code gen, ...)
- `{phase}` — human-readable phase name

**Example:**

```yaml
# .reviews/r01-p1-spec-review.yaml

review:
  module: order-cancel
  phase: spec-review
  round: 1
  reviewer: senior
  timestamp: 2026-02-12T10:30:00Z

findings:
  - id: F001
    severity: critical    # critical | high | medium | low
    category: schema-compliance
    description: "Missing error type for concurrent cancellation"
    location: spec.yaml#edge_cases
    suggestion: "Add ConcurrentCancelError to edge_cases"

summary:
  critical: 1
  high: 0
  medium: 0
  verdict: blocked        # pass | blocked | conditional
```

### 4.3 Rules

- **Agents produce only the `findings` list.** The orchestrator (or human) wraps them in review structure, auto-generates IDs, and computes summaries.
- **Agents never search for review files.** The orchestrator tells each agent exactly which reviews to read.
- **Invalid output triggers one re-prompt, then escalation.** Don't retry indefinitely.

---

## 5. Orchestration

### 5.1 Orchestrator Role

The orchestrator is the harness's central controller.
It can be a script, a human, or a higher-tier agent.

Responsibilities:

1. **Pre-compute the file list** each agent needs (from `.briefing.yaml` context hints)
2. **Assemble the handoff payload** — file contents, not file paths
3. **Validate outputs** against schemas
4. **Manage status** — agents never write status themselves
5. **Auto-generate review metadata** — IDs, timestamps, summaries
6. **Enforce gating logic** — critical findings block phase progression
7. **Track convergence** — round count, spec revisions, resolution status

### 5.2 Handoff Payload

What the orchestrator passes to each agent at phase start:

```yaml
handoff:
  module: order-cancel
  phase: senior-review
  round: 1
  spec_revision: 3
  files:
    spec: specs/order/cancel/spec.yaml
    code: specs/order/cancel/internals/cancel-order.ts
    reviews:
      - .reviews/r01-p1-spec-review.yaml
      - .reviews/r01-p2-impl-feedback.yaml
  previous_findings_summary:
    critical: 0
    high: 1
    spec_changes_applied: 2
  instructions: conventions/api.yaml
```

The agent receives this as initial context.
No filesystem navigation required.

### 5.3 Agent Context Loading Protocol

Every agent follows this protocol at session start:

```
Step 1: Receive handoff payload
        (module briefing + relevant files + previous findings)

Step 2: Read spec.yaml
        (the source of truth)

Step 3: Read task-specific context
        - Spec review:    governing schema (or summary)
        - Code gen:       target schemas + conventions
        - Senior review:  generated code + previous reviews
        - QA:             code + tests + all reviews

Step 4: Produce structured output
        - Findings list (YAML) for review phases
        - Code files for code gen phase
        - Test files for QA phase

Step 5: Return output for validation
```

The agent never decides what to load. The orchestrator decides.

### 5.4 Agent Roster Pattern

Map agents to MDA cycle phases. Each phase has a distinct agent role
to prevent bias (the agent that wrote code should not review it).

| Phase | Agent Role | Key Context |
|-------|------------|-------------|
| Spec Review | Senior | spec + governing schema |
| Code Generation | Junior | spec + target schemas + conventions |
| Senior Review | Reviewer (distinct from author) | spec + code + reviews |
| QA | QA Agent | spec + code + tests + reviews |

Missing agent roles should be identified early.
A phase without a defined agent role is a phase that cannot execute.

---

## 6. Conventions Directory

The conventions directory bridges the gap between framework-agnostic specs
and framework-specific code generation.

### 6.1 Structure

```
conventions/
  {target-a}.yaml     # Target-specific code gen rules
  {target-b}.yaml     # Target-specific code gen rules
  shared.yaml         # Common patterns across targets
  naming.yaml         # Naming conventions (files, exports, CSS classes, etc.)
```

### 6.2 Content Guidelines

Each file is a compact set of rules, not a tutorial.
An agent reads a convention file and knows:

- File structure and organization
- Naming patterns
- State management approach
- Export patterns
- Testing conventions
- Error handling patterns

**Keep convention files under 150 lines.** If longer, split by concern.

### 6.3 Example

```yaml
# conventions/react.yaml

file_patterns:
  component: "*.tsx"
  hook: "use*.ts"
  test: "*.test.tsx"

naming:
  components: PascalCase
  hooks: camelCase with 'use' prefix
  files: kebab-case

patterns:
  - "Extract component logic into custom hooks"
  - "Use composition over inheritance"
  - "Props interface named {Component}Props"

exports:
  - "Named exports only, no default exports"
  - "Re-export public API from index.ts"
```

---

## 7. Applying to Your Project

### Step 1: Measure Your Context

Before optimizing, measure:

1. Count lines in each manifest file (the map)
2. Count lines in each spec file (the territory)
3. Count lines in each schema file
4. Sum per-phase context requirements

### Step 2: Identify Bottlenecks

| Symptom | Cause | Solution |
|---------|-------|----------|
| Agents produce inconsistent output | Missing conventions | Create conventions/ files |
| Agents load irrelevant files | No briefings | Add `.briefing.yaml` per module |
| Code gen context too large | Full schemas loaded always | Add schema summaries |
| Phase handoffs lose context | No structured reviews | Introduce review.yaml artifacts |
| Agents search for files | No orchestration | Implement handoff payloads |

### Step 3: Prioritize

1. **Conventions** — without this, code gen agents cannot produce consistent output
2. **Review schema** — without this, agent output cannot be validated
3. **Schema summaries** — reduces heaviest agent loads
4. **Briefing generator** — auto-generates per-module briefings from manifests
5. **Orchestrator** — automates handoff payload assembly

Start with conventions. They have the highest impact-to-effort ratio.

---

## 8. Success Metrics

| Metric | Target | How to Measure |
|--------|--------|----------------|
| Map size | < 500 lines | Sum all manifest files |
| Agent context load (heaviest phase) | < 1,500 lines | Sum lines in handoff payload |
| Agent file discovery operations | 0 | Agents receive contents, never search |
| Review validation pass rate | > 90% first attempt | Valid / invalid output ratio |
| Average rounds per module | <= 2 | Round count from status tracking |

---

## 9. Glossary

| Term | Meaning |
|------|---------|
| **Harness** | Infrastructure (files, scripts, conventions) that enables agents to work reliably |
| **Briefing** | Auto-generated compact summary of what an agent needs for a specific module |
| **Context budget** | Maximum lines of source material an agent should load for a task |
| **Handoff payload** | File contents + metadata passed between agents across phases |
| **Tier 1 / 2 / 3** | Information loading tiers: briefing / essential / reference |
| **Map files** | The manifest.yaml chain — navigational structure |
| **Territory files** | spec.yaml, schema.yaml — actual content agents work with |
| **No Search** | Principle that agents receive all needed files, never search the filesystem |
| **Conventions** | Compact rule sets that bridge framework-agnostic specs and framework-specific code |

---

> Companion to the [MDA Playbook](./playbook.md).
> Based on patterns validated in the PDS-MDA project (2026-02-12).
