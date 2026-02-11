# CLAUDE.md

## Project Overview

woosgem-mda is a spec-driven multi-framework design system using Manifest-Driven Architecture (MDA).

**Core principle**: Specs are the source. Code is derived.

## Language Rules

- **README.md / README.en.md**: Korean and English respectively
- **Everything else**: English only
  - All spec files (spec.yaml)
  - All manifest files
  - All code comments
  - All commit messages
  - All documentation under docs/
  - All decision records (.decisions/)

## Architecture

- `manifest.yaml` — AI entry point (project map)
- `domains/atoms/` — 11 atomic components
- `domains/molecules/` — 9 molecule components
- `domains/organisms/` — 5 organism components
- `shared/` — tokens, CSP, utils, types
- `.conventions/` — code generation rules per framework

## Spec Structure

Each component has:
- `spec.yaml` — Core spec (framework-agnostic, web standards only)
- `spec.react.yaml` — React implementation spec (future)
- `spec.vue.yaml` — Vue implementation spec (future)
- `spec.lit.yaml` — Lit implementation spec (future)

## Key Conventions

- No component exists without a spec.yaml
- Spec uses web standard terms: attributes (not props), content (not slots), interactions (not events)
- Dependency direction: atom ← molecule ← organism (reverse forbidden)
- Cross-domain access only through contracts
