# WooSGem MDA

> Spec-driven Multi-framework Design System

MDA(Manifest-Driven Architecture)를 적용한 디자인 시스템. **스펙이 소스**이고, 코드는 파생물이다.

## 핵심 컨셉

```
spec.yaml (프레임워크 무관, 웹표준)
  ├── spec.react.yaml → React 컴포넌트
  ├── spec.vue.yaml   → Vue 컴포넌트
  ├── spec.lit.yaml   → Lit 웹컴포넌트
  └── spec.styles.yaml → SCSS 스타일
```

- **Spec-first**: spec.yaml 없는 컴포넌트는 존재할 수 없음
- **Framework-agnostic**: 코어 스펙은 웹표준 용어로만 기술 (props → attributes, slots → content, events → interactions)
- **Atomic Design**: 컴포넌트를 atom / molecule / organism으로 분류
- **Multi-framework**: 하나의 스펙에서 React, Vue, Lit 구현이 모두 나옴

## 구조

```
woosgem-mda/
  manifest.yaml              ← AI 진입점

  domains/
    atoms/                   ← 11개 (Button, Icon, Badge, ...)
    molecules/               ← 9개  (Input, Select, Checkbox, ...)
    organisms/               ← 5개  (Modal, Card, Tab, ...)

  shared/                    ← tokens, CSP, utils, types
  .conventions/              ← 프레임워크별 코드 생성 규칙
  .scripts/                  ← validate, generate
```

각 컴포넌트 디렉토리:

```
domains/atoms/button/
  spec.yaml              ← 코어 스펙 (웹표준, 프레임워크 무관)
  spec.react.yaml        ← React 구현 스펙
  spec.vue.yaml          ← Vue 구현 스펙
  spec.lit.yaml          ← Lit 구현 스펙
  spec.styles.yaml       ← 스타일 스펙
  spec.tests.yaml        ← 테스트 시나리오
  contracts/index.ts     ← public API
  internals/             ← 생성된 코드
```

## 컴포넌트

### Atoms

Button, IconButton, Icon, Badge, Avatar, Spinner, Skeleton, Kbd, Divider, Overlay, Progress

### Molecules

Input, Textarea, Select, Checkbox, Radio, Switch, Alert, Tooltip, ListItem

### Organisms

Modal, Card, Tab, SegmentedControl, Toast

## 배포

내부 조직은 도메인(atomic level) 기준이지만, 배포는 사용자 관점의 프레임워크별 패키지:

| 패키지 | 설명 |
|--------|------|
| `@woosgem-dev/core` | 프레임워크 무관 정의 + 타입 |
| `@woosgem-dev/react` | React 컴포넌트 |
| `@woosgem-dev/vue` | Vue 컴포넌트 |
| `@woosgem-dev/lit` | Lit 웹컴포넌트 |
| `@woosgem-dev/styles` | SCSS/CSS |

## 기술 스택

TypeScript, SCSS, React 18+, Vue 3.4+, Lit 3, Vite, Vitest, Turborepo

## 문서

- [설계 문서](docs/plans/2026-02-11-woosgem-mda-design.md)

## Status

**Phase 1**: 설계 진행 중
