# WooSGem MDA Design Document

> woosgem 디자인 시스템에 Manifest-Driven Architecture를 적용한 새 레포 설계

- **Date**: 2026-02-11
- **Status**: draft (브레인스토밍 진행 중)
- **Deciders**: WooSGem

---

## 1. 핵심 컨셉

### MDA + Atomic Design + Multi-Framework Design System

**MDA의 본질**: 코드를 짜기 전에 스펙을 상세하게 작성한다. **스펙이 소스**이고, 코드는 파생물이다.

**Atomic Design 도입**: 도메인을 atom / molecule / organism 3계층으로 분류한다.

**Multi-Framework**: 하나의 코어 스펙에서 React, Vue, Lit 구현이 모두 나온다.

```
spec.yaml (프레임워크 무관, 웹표준)
  ├── spec.react.yaml → React 코드
  ├── spec.vue.yaml   → Vue 코드
  ├── spec.lit.yaml   → Lit 코드
  └── spec.styles.yaml → SCSS
```

### 대전제

- **스펙이 소스, 코드는 파생물**: spec.yaml 없는 컴포넌트는 존재할 수 없음
- **프레임워크/언어에 묶이지 않는 스펙**: 코어 스펙은 웹표준 용어로만 기술
- **배포는 사용자 관점**: 내부는 도메인 기준, 배포는 기존대로 프레임워크별 패키지
- **패키지는 도메인보다 상위**: core/react/vue/lit은 도메인 안의 "레이어"

---

## 2. 디렉토리 구조

```
woosgem-mda/
  manifest.yaml                    # AI 진입점 (프로젝트 전체 맵)

  domains/
    atoms/                         # 도메인 1: 기본 단위 (11개)
      manifest.yaml                # 도메인 맵 + 전체 atom 목록
      button/
        spec.yaml                  # 코어 스펙 (프레임워크 무관)
        spec.react.yaml            # React 구현 스펙
        spec.vue.yaml              # Vue 구현 스펙
        spec.lit.yaml              # Lit 구현 스펙
        spec.styles.yaml           # 스타일 스펙
        spec.tests.yaml            # 테스트 시나리오
        contracts/index.ts         # 외부 공개 API
        internals/                 # 생성된 코드
          core/
          react/
          vue/
          lit/
          styles/
          tests/
      icon/
        spec.yaml
        ...
      # badge, avatar, spinner, skeleton, kbd, divider, overlay, progress, icon-button

    molecules/                     # 도메인 2: Atom 조합 (9개)
      manifest.yaml
      input/
        spec.yaml
        ...
      # textarea, select, checkbox, radio, switch, alert, tooltip, list-item

    organisms/                     # 도메인 3: 복합 구조 (5개)
      manifest.yaml
      modal/
        spec.yaml
        ...
      # card, tab, segmented-control, toast

  shared/                          # 2+ 도메인 공유
    manifest.yaml
    tokens/                        # 디자인 토큰
    csp/                           # Color Set Protocol
    utils/                         # 순수 유틸리티
    types/                         # 공통 타입

  .conventions/                    # 프레임워크별 코드 생성 규칙
    core.yaml
    react.yaml
    vue.yaml
    lit.yaml
    styles.yaml

  .decisions/                      # 아키텍처 결정 기록
  .scripts/                        # validate, generate 스크립트
```

---

## 3. 컴포넌트 분류

### Atoms (11개) — 최소 단위, 독립적

| 컴포넌트 | 설명 | Status |
|----------|------|--------|
| Button | 기본 인터랙션 버튼 | stable |
| IconButton | 아이콘 전용 버튼 | stable |
| Icon | SVG 아이콘 래퍼 | stable |
| Badge | 상태 표시 뱃지 | stable |
| Avatar | 사용자 아바타 | stable |
| Spinner | 로딩 인디케이터 | active |
| Skeleton | 로딩 플레이스홀더 | active |
| Kbd | 키보드 키 표시 | stable |
| Divider | 시각적 구분선 | stable |
| Overlay | 배경 오버레이 | stable |
| Progress | 진행률 바 | active |

### Molecules (9개) — Atom 조합, 단일 기능

| 컴포넌트 | 설명 | Status |
|----------|------|--------|
| Input | 텍스트 입력 | stable |
| Textarea | 멀티라인 입력 | stable |
| Select | 드롭다운 선택 | stable |
| Checkbox | 체크박스 | stable |
| Radio | 라디오 버튼 | stable |
| Switch | 토글 스위치 | stable |
| Alert | 알림 메시지 | stable |
| Tooltip | 호버/포커스 툴팁 | stable |
| ListItem | 리스트 아이템 | stable |

### Organisms (5개) — 복합 구조, 서브컴포넌트

| 컴포넌트 | 서브컴포넌트 | Status |
|----------|-------------|--------|
| Modal | Header, Body, Footer | stable |
| Card | Header, Body, Footer | stable |
| Tab | Tab, Panel | stable |
| SegmentedControl | Control, Item | stable |
| Toast | (positioning + auto-dismiss) | active |

---

## 4. Spec 구조

### 설계 원칙

- **`spec.yaml`** — "무엇"을 기술. 프레임워크/언어 무관. 웹표준 용어만
- **`spec.{framework}.yaml`** — "어떻게"를 기술. 프레임워크별 매핑 규칙
- 코어 스펙만으로 컴포넌트의 본질을 완전히 이해할 수 있어야 함

### 코어 스펙 용어 매핑

| 프레임워크 용어 | 코어 스펙 용어 | 이유 |
|---------------|--------------|------|
| props | **attributes** | HTML attribute가 웹표준 |
| slots / children | **content** (areas) | 프레임워크 중립 |
| events / handlers | **interactions** | 사용자 행위 관점 |
| portal / teleport | **rendering.strategy: portal** | 웹표준: top-layer |

### 4-1. 코어 스펙 — Button 예시

```yaml
# domains/atoms/button/spec.yaml
component:
  name: Button
  category: atom
  description: 기본 인터랙션 버튼

semantics:
  tag: button
  role: button

attributes:
  variant:
    type: enum
    values: [filled, outline, ghost, link]
    default: filled
    description: 시각적 스타일
  color:
    type: enum
    values: [primary, secondary, success, warning, danger, info, neutral]
    default: primary
  size:
    type: enum
    values: [xs, sm, md, lg]
    default: md
  loading:
    type: boolean
    default: false
  disabled:
    type: boolean
    default: false
  full-width:
    type: boolean
    default: false

content:
  default:
    description: 버튼 텍스트 또는 아이콘
    required: true

states:
  loading:
    visual: 스피너 표시, 텍스트 반투명
    behavior: 클릭 무시
    aria: { busy: true, disabled: true }
  disabled:
    visual: 흐리게 표시
    behavior: 클릭 무시, 포커스 가능
    aria: { disabled: true }

interactions:
  activate:
    triggers: [click, Enter, Space]
    condition: loading=false AND disabled=false
    description: 버튼 활성화 액션

accessibility:
  aria:
    busy: "loading 상태일 때 true"
    disabled: "disabled 상태일 때 true"
  keyboard:
    Enter: activate
    Space: activate
  focus: 기본 포커스 링 표시

visual_mapping:
  class: btn
  data_attributes:
    data-variant: variant
    data-color: color
    data-size: size
    data-state: "loading > disabled > undefined (우선순위)"
    data-full-width: "full-width일 때 존재"
```

### 4-2. React 구현 스펙

```yaml
# domains/atoms/button/spec.react.yaml
extends: ./spec.yaml

framework: react
target: ">=18"

component:
  export: named
  ref: HTMLButtonElement

props_mapping:
  variant: { name: variant, type: ButtonVariant }
  color: { name: color, type: ButtonColor }
  size: { name: size, type: ButtonSize }
  loading: { name: loading, type: boolean }
  disabled: { name: disabled, type: boolean }
  full-width: { name: fullWidth, type: boolean }
  native: Omit<ComponentPropsWithoutRef<'button'>, 'color'>

content_mapping:
  default: children

event_mapping:
  activate: onClick

implementation:
  factory: createComponent(ButtonDef)
  internal_deps: [PrefixContext, useId]
```

### 4-3. Vue 구현 스펙

```yaml
# domains/atoms/button/spec.vue.yaml
extends: ./spec.yaml

framework: vue
target: ">=3.4"

component:
  export: named
  define: DefineComponent<ButtonProps>

props_mapping:
  variant: { name: variant, type: "String as PropType<ButtonVariant>" }
  color: { name: color, type: "String as PropType<ButtonColor>" }
  size: { name: size, type: "String as PropType<ButtonSize>" }
  loading: { name: loading, type: Boolean }
  disabled: { name: disabled, type: Boolean }
  full-width: { name: fullWidth, type: Boolean }

content_mapping:
  default: "<slot />"

event_mapping:
  activate: "@click"

implementation:
  factory: createComponent(ButtonDef)
```

### 4-4. 서브컴포넌트 — Modal 예시 (코어 스펙)

```yaml
# domains/organisms/modal/spec.yaml
component:
  name: Modal
  category: organism
  description: 오버레이 다이얼로그. Header/Body/Footer 영역 구성

semantics:
  tag: div
  role: dialog

attributes:
  size:
    type: enum
    values: [sm, md, lg, full]
    default: md
  open:
    type: boolean
    default: false
    description: 모달 표시 여부 (제어 컴포넌트)
  close-on-overlay:
    type: boolean
    default: true
  close-on-esc:
    type: boolean
    default: true

composition:
  regions:
    header:
      tag: header
      description: 제목 영역
      required: false
    body:
      tag: div
      description: 본문 영역
      required: true
    footer:
      tag: footer
      description: 액션 영역
      required: false
  rendering:
    strategy: portal
    description: 문서 최상위에 렌더링하여 z-index 충돌 방지
  dependencies:
    atoms: [Overlay]

content:
  default:
    description: Header/Body/Footer 영역 배치
    required: true

states:
  open:
    visual: 모달 + 오버레이 표시
    behavior:
      - body scroll 잠금
      - 포커스 트랩 활성화
      - 첫 번째 focusable 요소로 포커스 이동
    aria: { modal: true }
  closed:
    visual: 렌더링되지 않음
    behavior:
      - body scroll 복원
      - 포커스 트랩 해제
      - 트리거 요소로 포커스 반환

interactions:
  close:
    triggers:
      - Escape 키 (close-on-esc=true일 때)
      - 오버레이 클릭 (close-on-overlay=true일 때)
      - 닫기 버튼 클릭
    description: 모달 닫기

accessibility:
  aria:
    modal: "true"
    labelledby: "Header 존재 시 자동 연결"
  keyboard:
    Escape: close
    Tab: "포커스 트랩 — 모달 내부에서만 순환"
  focus: 열릴 때 첫 focusable 요소, 닫힐 때 트리거로 복원

visual_mapping:
  class: modal
  data_attributes:
    data-size: size
    data-state: "open ? 'open' : 'closed'"
```

### 4-5. Modal React 구현 스펙

```yaml
# domains/organisms/modal/spec.react.yaml
extends: ./spec.yaml

framework: react
target: ">=18"

component:
  export: named
  ref: HTMLDivElement
  pattern: compound  # Modal.Header, Modal.Body, Modal.Footer

composition_mapping:
  regions:
    header: Modal.Header
    body: Modal.Body
    footer: Modal.Footer
  rendering:
    portal: createPortal(children, document.body)

props_mapping:
  size: { name: size, type: ModalSize }
  open: { name: open, type: boolean }
  close-on-overlay: { name: closeOnOverlay, type: boolean }
  close-on-esc: { name: closeOnEsc, type: boolean }

content_mapping:
  default: children

event_mapping:
  close: onClose

implementation:
  hooks: [useEffect, useCallback, useRef]
  internal_deps: [PrefixContext, useId]
  effects:
    - "scroll lock on open"
    - "focus trap on open"
    - "keydown listener for Escape"
```

### 4-6. Modal Vue 구현 스펙

```yaml
# domains/organisms/modal/spec.vue.yaml
extends: ./spec.yaml

framework: vue
target: ">=3.4"

component:
  export: named
  define: DefineComponent<ModalProps>
  pattern: slots  # named slots

composition_mapping:
  regions:
    header: "<slot name='header'>"
    body: "<slot name='default'>"
    footer: "<slot name='footer'>"
  rendering:
    portal: "<Teleport to='body'>"
    fallback: "teleportTo: false → Teleport 없이 렌더링"

props_mapping:
  size: { name: size, type: "String as PropType<ModalSize>" }
  open: { name: open, type: Boolean }
  close-on-overlay: { name: closeOnOverlay, type: Boolean }
  close-on-esc: { name: closeOnEsc, type: Boolean }
  teleport-to: { name: teleportTo, type: "[String, Boolean]", default: "body" }

content_mapping:
  default: "<slot />"

event_mapping:
  close: "@close"
```

---

## 5. Root Manifest

```yaml
# manifest.yaml
project:
  name: woosgem
  description: Multi-framework Design System — Spec-driven
  version: 0.1.0
  phase: 1  # 1: source+spec 공존 → 2: spec-first 전환 완료

stack:
  languages: [typescript, scss]
  frameworks: [react-18, vue-3, lit-3]
  bundler: vite
  test: vitest
  build_orchestrator: turborepo

domains:
  atoms:
    purpose: 최소 단위 UI 요소 — 단독으로 의미를 가지는 기본 블록
    status: active
    depends_on: []
    manifest: domains/atoms/manifest.yaml
    component_count: 11
  molecules:
    purpose: Atom 조합으로 단일 기능을 수행하는 요소
    status: active
    depends_on: [atoms]
    manifest: domains/molecules/manifest.yaml
    component_count: 9
  organisms:
    purpose: 복합 구조, 서브컴포넌트를 포함하는 고수준 요소
    status: active
    depends_on: [atoms, molecules]
    manifest: domains/organisms/manifest.yaml
    component_count: 5

shared:
  path: shared/
  manifest: shared/manifest.yaml
  modules:
    tokens: 디자인 토큰 (spacing, typography, color primitives)
    csp: Color Set Protocol — 최소 입력으로 81개 토큰 생성
    utils: 공유 유틸리티 (2+ 도메인 사용)
    types: 공통 타입 정의

publish:
  strategy: framework-based
  packages:
    "@woosgem-dev/core": 프레임워크 무관 정의 + 타입
    "@woosgem-dev/react": React 래퍼 전체
    "@woosgem-dev/vue": Vue 래퍼 전체
    "@woosgem-dev/lit": Lit 웹컴포넌트 전체
    "@woosgem-dev/styles": SCSS/CSS 번들

conventions:
  - .conventions/core.yaml
  - .conventions/react.yaml
  - .conventions/vue.yaml
  - .conventions/lit.yaml
  - .conventions/styles.yaml

validation:
  script: .scripts/validate-manifest.ts
  checks:
    - spec.yaml 존재 여부
    - spec 필수 필드 완전성
    - spec ↔ code 동기화
    - contracts export 매칭
    - 의존성 방향 (atom ← molecule ← organism)
    - 도메인 간 internals 직접 import 금지
    - shared 사용 빈도 (2+ 도메인)
    - 미선언 파일 탐지
    - 컨벤션 준수

rules:
  - spec.yaml 없는 컴포넌트는 존재할 수 없음
  - 도메인 간 통신은 contracts만 허용
  - organism → atom 직접 의존 가능 (molecule 거칠 필요 없음)
  - 의존성 역방향 금지 (atom이 molecule을 참조할 수 없음)
  - shared에는 2+ 도메인이 쓰는 코드만
```

---

## 6. Domain Manifest 예시 — Atoms

```yaml
# domains/atoms/manifest.yaml
domain:
  name: atoms
  purpose: 최소 단위 UI 요소
  status: active
  depends_on: []

components:
  button:
    name: Button
    status: stable
    spec: button/spec.yaml
    contracts: button/contracts/index.ts
  icon-button:
    name: IconButton
    status: stable
    spec: icon-button/spec.yaml
    depends_on: [button, icon]
  icon:
    name: Icon
    status: stable
    spec: icon/spec.yaml
  badge:
    name: Badge
    status: stable
    spec: badge/spec.yaml
  avatar:
    name: Avatar
    status: stable
    spec: avatar/spec.yaml
  spinner:
    name: Spinner
    status: active
    spec: spinner/spec.yaml
  skeleton:
    name: Skeleton
    status: active
    spec: skeleton/spec.yaml
  kbd:
    name: Kbd
    status: stable
    spec: kbd/spec.yaml
  divider:
    name: Divider
    status: stable
    spec: divider/spec.yaml
  overlay:
    name: Overlay
    status: stable
    spec: overlay/spec.yaml
  progress:
    name: Progress
    status: active
    spec: progress/spec.yaml

contracts:
  path: contracts/index.ts
  description: 전체 atoms 도메인의 public API
  exports:
    types:
      - ButtonVariant, ButtonColor, ButtonSize
      - IconSize, IconColor
      - BadgeVariant, BadgeColor, BadgeSize
    components:
      react: [Button, IconButton, Icon, Badge, Avatar, Spinner, Skeleton, Kbd, Divider, Overlay, Progress]
      vue: [Button, IconButton, Icon, Badge, Avatar, Spinner, Skeleton, Kbd, Divider, Overlay, Progress]
      lit: [wg-button, wg-icon-button, wg-icon, wg-badge, wg-avatar, wg-spinner, wg-skeleton, wg-kbd, wg-divider, wg-overlay, wg-progress]
```

---

## 7. Shared 도메인

```yaml
# shared/manifest.yaml
shared:
  purpose: 2+ 도메인이 사용하는 공통 코드

  modules:
    tokens:
      purpose: 디자인 토큰 — spacing, typography, color primitives
      phase: permanent
      spec: tokens/spec.yaml
      files:
        - tokens/spacing.ts
        - tokens/typography.ts
        - tokens/colors.ts
        - tokens/breakpoints.ts

    csp:
      purpose: Color Set Protocol — 최소 색상 입력 → 81개 토큰 자동 생성
      phase: permanent
      spec: csp/spec.yaml
      files:
        - csp/generator.ts
        - csp/transformers.ts
        - csp/validators.ts
        - csp/types.ts

    utils:
      purpose: 순수 유틸리티 함수
      phase: phase-2-candidate
      spec: utils/spec.yaml
      files:
        - utils/class-names.ts
        - utils/merge-props.ts

    types:
      purpose: 크로스 도메인 타입 정의
      phase: phase-2-candidate
      files:
        - types/component-definition.ts
        - types/style-props.ts

  rules:
    - 단일 도메인만 사용하는 코드는 해당 도메인으로 이동
    - utils는 순수 함수만 (부수 효과 금지)
    - tokens 변경 시 모든 도메인에 영향 — 반드시 .decisions/ 기록
```

---

## 8. Validation 규칙

### 검증 체크리스트 (9개)

| # | 검증 항목 | 심각도 | 설명 |
|---|----------|--------|------|
| 1 | spec 존재 | ERROR | 모든 컴포넌트에 spec.yaml 존재 |
| 2 | spec 필수 필드 | ERROR | component, semantics, attributes, content, states, accessibility, visual_mapping |
| 3 | spec ↔ code 동기화 | ERROR | spec attributes ↔ core mapPropsToAttrs ↔ framework props |
| 4 | contracts export 매칭 | ERROR | contracts/index.ts exports = manifest 선언 |
| 5 | 의존성 방향 | ERROR | atom←molecule←organism, 역방향 금지 |
| 6 | internals 직접 import 금지 | ERROR | 도메인 간 contracts만 허용 |
| 7 | shared 사용 빈도 | WARNING | 1개 도메인만 사용 → 이동 권고 |
| 8 | 미선언 파일 | WARNING | manifest에 없는 소스 파일 탐지 |
| 9 | 컨벤션 준수 | WARNING | .conventions/ 규칙 준수 여부 |

### 기존 MDA 대비 추가

| 기존 MDA (B2B 앱) | woosgem MDA (디자인 시스템) |
|---|---|
| 파일 존재 | **spec 존재** (spec 없으면 컴포넌트 존재 불가) |
| export 매칭 | **spec ↔ code 동기화** (스펙이 소스이므로) |
| import 경계 | import 경계 + **의존성 방향** (atom→molecule→organism) |
| shared 사용 빈도 | shared 사용 빈도 |
| 미선언 파일 | 미선언 파일 |
| 의존성 일관성 | **컨벤션 준수** (프레임워크별 코드 규칙) |

---

## 9. 남은 설계 항목 (TODO)

- [ ] 코드 생성 플로우 — spec에서 코드가 어떻게 나오는지
- [ ] 빌드/배포 파이프라인 — 도메인 구조 → 프레임워크별 패키지 번들
- [ ] .conventions/ 상세 — 프레임워크별 코드 생성 규칙
- [ ] 마이그레이션 전략 — 기존 woosgem 코드를 어떻게 옮기는지
- [ ] Agent 역할 분담 — spec 작성은 누가, 코드 생성은 누가

---

## 10. 참조

- [MDA Reference Project](../../manifest-driven-architecture/README.md)
- [기존 woosgem Design System](../../woosgem/)
- [MDA Blueprint](../../manifest-driven-architecture/reference/blueprint.md)
