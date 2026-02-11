# Composable Schema System

Schemas are composable building blocks. Every spec.yaml combines one or more schemas via `$schema`.

## Directory Structure

```
schemas/
  base/         — Functional specs (WHAT it does, language-agnostic)
  lang/         — Language-specific type systems and signatures
  platform/     — Platform-specific rendering and accessibility
  framework/    — Framework-specific implementation patterns
```

## Composition Formula

```
spec = base + lang [+ platform] [+ framework]
```

### Shared modules (tokens, CSP, utils, types)

```yaml
# shared/utils/spec.yaml — behavior and test cases only
$schema: schemas/base/function.schema.yaml

# shared/utils/spec.ts.yaml — TypeScript signatures
$schema:
  compose: [schemas/base/function.schema.yaml, schemas/lang/typescript.schema.yaml]

# shared/utils/spec.kt.yaml — Kotlin signatures (future)
$schema:
  compose: [schemas/base/function.schema.yaml, schemas/lang/kotlin.schema.yaml]
```

### Component specs

```yaml
# domains/atoms/button/spec.yaml — what Button IS
$schema: schemas/base/component.schema.yaml

# domains/atoms/button/spec.web.yaml — web platform mapping
$schema:
  compose: [schemas/base/component.schema.yaml, schemas/platform/web.schema.yaml]

# domains/atoms/button/spec.react.yaml — React implementation
$schema:
  compose:
    - schemas/base/component.schema.yaml
    - schemas/platform/web.schema.yaml
    - schemas/framework/react.schema.yaml
    - schemas/lang/typescript.schema.yaml

# domains/atoms/button/spec.ios.yaml — iOS/SwiftUI implementation
$schema:
  compose:
    - schemas/base/component.schema.yaml
    - schemas/platform/ios.schema.yaml
    - schemas/lang/swift.schema.yaml

# domains/atoms/button/spec.compose.yaml — Android/Compose implementation
$schema:
  compose:
    - schemas/base/component.schema.yaml
    - schemas/platform/compose.schema.yaml
    - schemas/lang/kotlin.schema.yaml
```

## Rules

1. `base/` schemas are always required (exactly one per spec file)
2. `lang/` schemas add type signatures, generics, and language-specific syntax
3. `platform/` schemas add rendering, accessibility, and interaction mapping
4. `framework/` schemas add implementation patterns (hooks, composables, etc.)
5. A spec file with only `$schema: string` uses a single schema (no composition)
6. A spec file with `$schema.compose: array` merges multiple schemas
7. Later schemas in the compose array can override fields from earlier ones
8. `base/` spec.yaml files must contain NO language-specific syntax

## Future: Framework Sub-Patterns

Framework schemas may be split into sub-patterns when needed:

```
framework/
  react.schema.yaml              # Current: shared React conventions
  # react-component.schema.yaml  # Future: component pattern (JSX, forwardRef)
  # react-hook.schema.yaml       # Future: hook pattern (use*, rules of hooks)
  # react-context.schema.yaml    # Future: context pattern (Provider, useContext)
```
