# WooSGem MDA

**[한국어](./README.md)** | English

> Spec-driven Multi-framework Design System

A design system powered by MDA (Manifest-Driven Architecture). **Specs are the source**, code is derived.

## Core Concept

```
spec.yaml (framework-agnostic, web standards)
  ├── spec.react.yaml → React components
  ├── spec.vue.yaml   → Vue components
  ├── spec.lit.yaml   → Lit web components
  └── spec.styles.yaml → SCSS styles
```

- **Spec-first**: No component can exist without a spec.yaml
- **Framework-agnostic**: Core specs use only web standard terminology (props → attributes, slots → content, events → interactions)
- **Atomic Design**: Components classified as atom / molecule / organism
- **Multi-framework**: A single spec produces React, Vue, and Lit implementations

## Structure

```
woosgem-mda/
  manifest.yaml              ← AI entry point

  specs/
    atoms/                   ← 11 (Button, Icon, Badge, ...)
    molecules/               ← 9  (Input, Select, Checkbox, ...)
    organisms/               ← 5  (Modal, Card, Tab, ...)

  shared/                    ← tokens, CSP, utils, types
  conventions/               ← Framework-specific code generation rules
  workers/                   ← Agent workspaces
  decisions/                 ← Architecture decision records
  config/                    ← Build/publish config
  scripts/                   ← validate, generate
```

Each component directory:

```
specs/atoms/button/
  spec.yaml              ← Core spec (web standards, framework-agnostic)
  spec.web.yaml          ← Web platform mapping
  spec.react.yaml        ← React implementation spec
  spec.vue.yaml          ← Vue implementation spec
  spec.lit.yaml          ← Lit implementation spec
```

## Components

### Atoms

Button, IconButton, Icon, Badge, Avatar, Spinner, Skeleton, Kbd, Divider, Overlay, Progress

### Molecules

Input, Textarea, Select, Checkbox, Radio, Switch, Alert, Tooltip, ListItem

### Organisms

Modal, Card, Tab, SegmentedControl, Toast

## Distribution

Internal organization is by domain (atomic level), but distribution is framework-based packages from user perspective:

| Package | Description |
|---------|-------------|
| `@woosgem-dev/core` | Framework-agnostic definitions + types |
| `@woosgem-dev/react` | React components |
| `@woosgem-dev/vue` | Vue components |
| `@woosgem-dev/lit` | Lit web components |
| `@woosgem-dev/styles` | SCSS/CSS |

## Tech Stack

TypeScript, SCSS, React 18+, Vue 3.4+, Lit 3, Vite, Vitest, Turborepo

## Documentation

- [MDA Playbook](docs/playbook.md)
- [Harness Engineering](docs/harness-engineering.md)
- [Design Document](docs/plans/2026-02-11-woosgem-mda-design.md)

## Status

**Phase 1**: Design in progress
