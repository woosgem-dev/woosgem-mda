# MDA Playbook

> AI agent as first-class citizen in project architecture.
> Human is team lead, AI is practitioner. Manifest is the map.

A practical guide for starting new projects with MDA.
Based on patterns validated in the PDS-MDA project.

---

## Table of Contents

1. [Core Principles](#1-core-principles)
2. [Project Structure](#2-project-structure)
3. [Manifest Patterns](#3-manifest-patterns)
4. [Worker Hierarchy](#4-worker-hierarchy)
5. [Spec System](#5-spec-system)
6. [MDA Development Cycle](#6-mda-development-cycle)
7. [Quick Start](#7-quick-start)
8. [Domain Type Examples](#8-domain-type-examples)
9. [Principle Summary](#9-principle-summary)

---

## 1. Core Principles

### Manifest = Route

A manifest is a routing table that declares "what exists here."
AI agents read the manifest before code. Anything not in the manifest does not exist.

- Root manifest -> config paths + domain paths + workers path
- Domain manifest -> child spec/module paths
- All manifests follow the **structure + rules** pattern

### On-Demand Navigation

AI reads manifests hierarchically, only as much as needed.

```
1. root manifest.yaml    -> understand overall structure
2. workers/              -> check assigned work
3. relevant domain only  -> identify work target
4. read code             -> minimum scope
```

Even with 30 domains, the irrelevant 28 are never read.

### Explicit Over Implicit

Zero implicit conventions.
All rules are declared in manifests or conventions.
AI does not guess; it follows only declared rules.

### AI Autonomy within Boundaries

AI autonomously decides structure within its authority.
Authority is clearly separated by worker tier (senior / mid / junior).

---

## 2. Project Structure

```
project/
├── manifest.yaml              # Entry point (structure + rules)
│
├── schemas/                   # Schemas (compiler rules)
│   ├── manifest.yaml
│   └── *.schema.yaml
│
├── conventions/               # Repeatable rules
│   └── *.yaml
│
├── decisions/                 # ADR (decision records)
│   └── YYYY-MM-DD-*.md
│
├── workers/                   # Agent workspaces
│   ├── manifest.yaml
│   ├── senior/
│   │   ├── tier.yaml
│   │   └── {worker-name}/
│   ├── mid/
│   │   ├── tier.yaml
│   │   └── {worker-name}/
│   └── junior/
│       ├── tier.yaml
│       └── {worker-name}/
│
└── specs/                     # Domain specs (Source of Truth)
    ├── {domain-a}/
    │   ├── manifest.yaml
    │   └── {module}/spec.yaml
    └── {domain-b}/
        ├── manifest.yaml
        └── {module}/spec.yaml
```

### Directory Naming Rules

- **No dot-prefix** -- AI must discover directories easily
- conventions, decisions, workers are all regular directories
- Domains are freely defined by business or system boundary

---

## 3. Manifest Patterns

All manifests consist of two sections: **structure + rules**.

### Root Manifest

```yaml
# manifest.yaml -- Project entry point

project:
  name: my-project
  description: "One-line project description"

config:
  schemas: schemas/
  conventions: conventions/
  decisions: decisions/

workers: workers/

domains:
  domain-a: specs/domain-a/
  domain-b: specs/domain-b/

rules:
  - "Project-level mandatory rule 1"
  - "Project-level mandatory rule 2"
```

### Domain Manifest

```yaml
# specs/{domain}/manifest.yaml

structure:
  module-a: module-a/spec.yaml
  module-b: module-b/spec.yaml

rules:
  - "Rules to follow in this domain"
```

### Config Manifest

Only for complex config directories (e.g., schemas).

```yaml
# schemas/manifest.yaml

structure:
  core:
    component: component.schema.yaml
  framework:
    react: framework/react.schema.yaml

rules:
  - "Schema-related rules"
```

### Flat Config Directories

Simple directories like conventions/ and decisions/ don't need their own manifest.
The root manifest points to the path; AI discovers files via glob.

---

## 4. Worker Hierarchy

Three-tier authority system. Each tier can have multiple workers.

### Permission Matrix

|            | schemas/ | conventions/ | decisions/ | specs/ | generated/ |
| ---------- | -------- | ------------ | ---------- | ------ | ---------- |
| **Senior** | R/W      | R/W          | R/W        | R/W    | R          |
| **Mid**    | R        | R            | R          | R/W    | R          |
| **Junior** | R        | R            | R          | R      | W          |

### Role Definitions

**Senior** -- Architect
- Schema/convention change authority
- Spec review approval, spec finalization
- ADR authoring
- Task assignment (to lower tiers)

**Mid** -- Spec Author
- Spec creation/modification (until review)
- Domain manifest management
- Review feedback incorporation

**Junior** -- Code Generator
- Mechanically derive code from finalized specs
- Spec read-only (no modification)

### Worker File Structure

```yaml
# workers/senior/tier.yaml

tier: senior
permissions:
  schemas: [read, write]
  conventions: [read, write]
  decisions: [read, write]
  specs: [read, write]
```

```yaml
# workers/senior/{worker-name}/agent.yaml

name: architect-1
tier: senior
model: claude-opus-4-6
status: idle               # idle | working | blocked
current_task: ~
assigned_domains: [shared, atoms]
```

### Assignment Rules

- Only higher tiers can assign tasks to lower tiers
- No task assignment between same tiers
- No authority escalation between tiers

---

## 5. Spec System

### Spec = Source of Truth

spec.yaml is the single source. Code is mechanically derived from specs.
Never add to code what isn't in the spec.

### Spec Status (Cycle Position)

Each spec declares its MDA cycle stage:

```yaml
# Top of each spec.yaml
spec_status: draft    # draft -> review -> stable -> codegen
```

| Status    | Meaning             | Allowed Actions                        |
| --------- | ------------------- | -------------------------------------- |
| `draft`   | Initial draft       | Mid freely modifies                    |
| `review`  | Awaiting/in review  | Senior reviews, Mid incorporates feedback |
| `stable`  | Finalized           | Changes require Senior approval        |
| `codegen` | Code generated      | Spec changes require regeneration      |

### Pre-Work Checklist

Before starting any work, always verify:

1. **MDA cycle stage** -- Is `spec_status` appropriate for the current work?
2. **Dependency chain** -- Do referenced specs actually exist?
3. **Schema changes** -- Schema changes only happen in the retrospective stage

---

## 6. MDA Development Cycle

```
1. Review feature requirements  -> Analyze PRD, Figma, Jira, etc.
2. Create spec                  -> Write spec.yaml guided by schema
3. Review spec                  -> Multi-angle verification (schema compliance, Figma match, dependencies)
4. Revise spec                  -> Incorporate review feedback
5. Iterate (2 -> 3 -> 4)       -> Until desired quality reached
6. Finalize & retrospective     -> Finalize spec, discuss schema changes, record ADR
7. Generate code                -> Mechanically derive from finalized spec
```

### Worker <-> Cycle Mapping

| Cycle Stage                   | Responsible Worker |
| ----------------------------- | ------------------ |
| 1. Review feature requirements | Senior             |
| 2. Create spec                 | Mid                |
| 3. Review spec                 | Senior             |
| 4. Revise spec                 | Mid                |
| 5. Iterate                     | Mid + Senior       |
| 6. Finalize / retrospective    | Senior             |
| 7. Generate code               | Junior             |

### Anti-Patterns

| Pattern                    | Why It Fails                             | Correct Approach                     |
| -------------------------- | ---------------------------------------- | ------------------------------------ |
| Spec + code simultaneously | Code must be fixed when spec changes     | Generate code after spec is finalized |
| Schema modified after fact | Schema becomes a record, not a rule      | Change only in stage 6 (retrospective) |
| Skip review, proceed fully | Errors propagate to all targets          | Iterative module-level reviews       |

---

## 7. Quick Start

### Step 0: .agentignore

Declare paths AI agents should not explore.
Like .gitignore is for git, .agentignore is for AI.

Typical exclusions:
- IDE settings (.idea/, .vscode/)
- Dependencies (node_modules/)
- Build artifacts (dist/, build/)
- Environment variables (.env)
- Lock files (pnpm-lock.yaml, etc.)

Manifest already ignores unlisted items, but explicit exclusion in .agentignore prevents token waste during glob/exploration.

### Step 1: Directory Structure

```
project/
├── .agentignore
├── manifest.yaml
├── schemas/
├── conventions/
├── decisions/
├── workers/
│   ├── senior/
│   ├── mid/
│   └── junior/
└── specs/
    ├── {domain-a}/
    └── {domain-b}/
```

Domains are freely defined to match project characteristics.

### Step 2: Root Manifest

```yaml
# manifest.yaml

project:
  name: my-project
  description: "One-line project description"

config:
  schemas: schemas/
  conventions: conventions/
  decisions: decisions/

workers: workers/

domains:
  domain-a: specs/domain-a/
  domain-b: specs/domain-b/

rules:
  - "Mandatory rules for this project"
```

### Step 3: Workers Manifest

```yaml
# workers/manifest.yaml

structure:
  senior: senior/
  mid: mid/
  junior: junior/

rules:
  - "Only higher tiers can assign tasks to lower tiers"
  - "Senior: schema/convention/ADR changes, review approval"
  - "Mid: spec creation/modification"
  - "Junior: code generation"
  - "No authority escalation between tiers"
```

### Step 4: Domain Manifests

```yaml
# specs/{domain}/manifest.yaml

structure: {}

rules:
  - "Domain rules"
```

### Step 5: First Spec

Follow the MDA cycle:
Define schema -> Write spec -> Review -> Iterate revisions -> Finalize -> Generate code.

---

## 8. Domain Type Examples

MDA is not framework-dependent.
Domain definitions vary by project characteristics.

### Design System (PDS-MDA Case)

```yaml
domains:
  shared: specs/shared/       # tokens, utils
  atoms: specs/atoms/         # minimal unit components
  molecules: specs/molecules/ # atoms combinations
  organisms: specs/organisms/ # molecules combinations
```

- spec = component/token definition
- code generation = SCSS, TypeScript, React, Vue

### Business Application (General Case)

```yaml
domains:
  order: specs/order/         # order domain
  product: specs/product/     # product domain
  auth: specs/auth/           # auth domain
```

- spec = API contracts, business logic definitions
- code generation = API clients, types, stores

### Common Rules

Regardless of domain type:
- Manifests follow the **structure + rules** pattern
- Spec is the Source of Truth
- Authority separated by worker tier
- MDA cycle is followed

---

## 9. Principle Summary

| # | Principle                  | Description                                      |
| - | -------------------------- | ------------------------------------------------ |
| 1 | **Manifest = Route**       | Manifests only route; details live at each location |
| 2 | **Structure + Rules**      | Unified pattern for all manifests                |
| 3 | **Spec = Source of Truth** | Code is a derivative                             |
| 4 | **Worker Hierarchy**       | Senior / Mid / Junior three-tier authority        |
| 5 | **On-Demand Navigation**   | Read only the manifests you need                 |
| 6 | **Explicit Over Implicit** | Undeclared rules don't exist                     |
| 7 | **No Dot-Prefix**          | Easy for AI to discover                          |
| 8 | **Decision Memory**        | All decisions recorded in decisions/             |

---

> This document was written based on PDS-MDA project experience (2026-02-12).
> Not perfect; will continue to evolve through application to new projects.
