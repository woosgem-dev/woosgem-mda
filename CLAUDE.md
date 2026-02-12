# CLAUDE.md

## Project Overview

woosgem-mda is a spec-driven multi-framework design system using Manifest-Driven Architecture (MDA).

**Core principle**: Specs are the source. Code is derived.

## Language Rules

- **README.md / README.en.md**: Korean and English respectively
- **Everything else**: English only
  - All spec files (spec.yaml)
  - All schema files (schemas/*.schema.yaml)
  - All manifest files
  - All code comments
  - All commit messages
  - All documentation under docs/
  - All decision records (decisions/)

## Architecture

```
manifest.yaml                 — AI entry point (project map)
schemas/                      — Composable schema system
  base/                       — Layer 0: functional specs (WHAT it does)
    component.schema.yaml     — 25 component specs (merged Layer 0+1 for web)
    token.schema.yaml         — design token specs
    protocol.schema.yaml      — protocol specs (CSP)
    function.schema.yaml      — utility function specs
    type.schema.yaml          — type definition specs
  lang/                       — Language-specific type systems
    typescript.schema.yaml    — TS signatures, generics, interfaces
    kotlin.schema.yaml        — Kotlin data classes, sealed classes
    swift.schema.yaml         — Swift structs, enums, protocols
  platform/                   — Layer 1: platform mapping
    web.schema.yaml           — HTML, ARIA, CSS hooks, DOM events
    ios.schema.yaml           — SwiftUI, VoiceOver, ViewModifiers
    compose.schema.yaml       — Compose, Semantics, Modifier chain
    rn.schema.yaml            — React Native, StyleSheet, accessibility
  framework/                  — Layer 2: framework patterns (web only)
    react.schema.yaml         — JSX, hooks, forwardRef, Context
    vue.schema.yaml           — SFC, Composition API, v-model, provide/inject
    lit.schema.yaml           — Custom elements, Shadow DOM, CSS parts
specs/
  atoms/                      — 11 atomic components
  molecules/                  — 9 molecule components
  organisms/                  — 5 organism components
shared/                       — Cross-domain modules
  tokens/spec.yaml            — Design tokens (DTCG-compatible, platform-neutral)
  csp/spec.yaml               — Color Set Protocol (81 tokens from minimal input)
  utils/spec.yaml             — Pure utility functions (classNames, mergeProps, filterNullish)
  types/spec.yaml             — Shared type definitions (ComponentDefinition, token key unions)
conventions/                  — Code generation rules per framework
workers/                      — Agent workspaces (senior/mid/junior)
decisions/                    — Architecture decision records
config/                       — Build and validation config
```

## Spec + Schema System

Every spec.yaml declares its schema via `$schema` field at the top. Schemas are composable:

```
Composition formula: spec = base + lang [+ platform] [+ framework]
```

### Simple schema (single base)

```yaml
$schema: schemas/base/component.schema.yaml   # Layer 0 — what it IS
$schema: schemas/base/token.schema.yaml       # token modules
$schema: schemas/base/protocol.schema.yaml    # protocol modules (CSP)
$schema: schemas/base/function.schema.yaml    # function modules
$schema: schemas/base/type.schema.yaml        # type modules
```

### Composed schema (multiple layers)

```yaml
# spec.react.yaml — React component implementation
$schema:
  compose:
    - schemas/base/component.schema.yaml       # Layer 0
    - schemas/platform/web.schema.yaml         # Layer 1
    - schemas/framework/react.schema.yaml      # Layer 2
    - schemas/lang/typescript.schema.yaml      # Language

# spec.ios.yaml — iOS/SwiftUI implementation
$schema:
  compose:
    - schemas/base/component.schema.yaml
    - schemas/platform/ios.schema.yaml
    - schemas/lang/swift.schema.yaml
```

### Layered spec structure

- **Layer 0**: `spec.yaml` — platform-agnostic (what it IS)
- **Layer 1**: `spec.web.yaml` / `spec.ios.yaml` / `spec.compose.yaml` — platform mapping
- **Layer 2**: `spec.react.yaml` / `spec.vue.yaml` / `spec.lit.yaml` — framework implementation (web only)
- iOS and Android merge Layer 1+2 (SwiftUI/Compose ARE their frameworks)

## Shared Module Design

- **Platform-neutral**: Token values are unitless numbers. Units attached at build time per platform.
- **DTCG-compatible**: Token scales use `$type` annotations (dimension, duration, cubicBezier, etc.)
- **Multi-platform output**: Each token scale has `output_patterns` as an open map (keys are platform identifiers)
- **Structured DSL**: CSP transformations use `{ fn: name, args: [...] }` — no string formulas
- **Token derivation forms**: `{ input }`, `{ transform, source }`, `{ source }`, `{ default: { light, dark } }`

## Component Spec Files

Each component has layered spec files:
- `spec.yaml` — Layer 0: platform-agnostic (what it IS)
- `spec.web.yaml` — Layer 1: web platform mapping (HTML, ARIA, CSS, DOM events)
- `spec.react.yaml` — Layer 2: React implementation (future)
- `spec.vue.yaml` — Layer 2: Vue implementation (future)
- `spec.lit.yaml` — Layer 2: Lit implementation (future)
- `spec.ios.yaml` — Layer 1+2: iOS/SwiftUI (future)
- `spec.compose.yaml` — Layer 1+2: Android/Compose (future)
- `spec.rn.yaml` — Layer 1+2: React Native (future)

## Key Conventions

- No component exists without a spec.yaml
- Every spec.yaml must have a `$schema` declaration
- Layer 0 spec uses web-standard terms: attributes, content, interactions, semantics, accessibility, visual_mapping
- spec.yaml = Layer 0 + Layer 1 merged for web (per "Accept the Merge" decision)
- Dependency direction: atom ← molecule ← organism (reverse forbidden)
- Cross-domain access only through contracts
- Shared code must be used by 2+ domains
- All shared functions must be pure (no side effects)
- Token changes affect all domains — require decisions/ record
