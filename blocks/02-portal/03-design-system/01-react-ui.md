---
id: ds-react-ui
category: design-system
label: React 네이티브 컴포넌트
summary: Design Token 기반 위에 React 네이티브 컴포넌트 라이브러리(@portal/react-ui)를 구축하여 최적의 개발 경험을 제공한다.
pros:
  - React의 타입 추론, props, hooks와 완벽하게 통합되어 최고의 DX(개발자 경험)
  - 상태 관리, 이벤트 핸들링, 조건부 렌더링 등이 React 패턴 그대로 사용 가능
  - 트리 셰이킹으로 사용하지 않는 컴포넌트는 번들에서 자동 제외
  - Storybook, Testing Library 등 React 생태계의 도구를 그대로 활용 가능
cons:
  - React가 아닌 앱(Vue, Angular 등)에서는 사용 불가
  - React 메이저 버전 업데이트 시 컴포넌트 라이브러리도 함께 업데이트 필요
  - 이기종 프레임워크 앱이 추가될 경우 별도 대응 필요 (Web Components fallback 등)
bestFor:
  - 모든 또는 대부분의 Remote App이 React로 개발되는 프로젝트
  - React 생태계(Next.js, Remix 등)를 주력으로 사용하는 조직
  - 컴포넌트 수준의 일관성과 높은 개발 생산성이 중요한 경우
---

## 개요

React 네이티브 컴포넌트 전략은 Design Token(Tier 1) 위에 React 전용 UI 컴포넌트 라이브러리(Tier 2)를 구축하는 방식이다. `@portal/react-ui` 패키지로 배포되며, 모든 React 기반 Remote App이 이를 import하여 일관된 UI를 구현한다.

이 방식의 핵심 가치는 **네이티브 개발 경험**이다. Web Components 같은 추상화 없이 React의 모든 기능(TypeScript 타입 추론, props 자동완성, hooks 통합, Context 연동)을 그대로 활용할 수 있다.

## 컴포넌트 라이브러리 구성

### 패키지 구조

```
@portal/design-tokens   → Tier 1: CSS 변수, JSON 토큰
@portal/react-ui        → Tier 2: React 컴포넌트 라이브러리
@portal/react-icons     → 아이콘셋
@portal/react-charts    → 차트 컴포넌트 (옵션)
```

### 컴포넌트 분류

| 분류 | 컴포넌트 | 설명 |
|------|---------|------|
| **기초(Primitive)** | Button, Input, Select, Checkbox, Radio, Toggle | 가장 기본적인 입력/액션 요소 |
| **레이아웃** | Stack, Grid, Flex, Divider, Spacer | 배치와 간격 제어 |
| **네비게이션** | Tabs, Breadcrumb, Pagination, Stepper | 화면 이동/단계 표시 |
| **데이터 표시** | Table, DataGrid, Card, Badge, Tag, Avatar | 데이터 시각화 |
| **피드백** | Toast, Alert, Modal, Drawer, Tooltip, Popover | 사용자 알림/안내 |
| **폼** | Form, FormField, DatePicker, FileUpload | 데이터 입력 그룹 |

### 컴포넌트 설계 원칙

1. **Composition 패턴**: 컴포넌트를 작은 단위로 분리하여 조합 가능하게 설계 (예: `<Table>` → `<Table.Head>`, `<Table.Body>`, `<Table.Row>`)
2. **Controlled/Uncontrolled 지원**: 모든 입력 컴포넌트는 제어/비제어 모드 모두 지원
3. **접근성(a11y) 내장**: WAI-ARIA 속성, 키보드 내비게이션, 스크린 리더 지원을 기본 포함
4. **Design Token 연동**: 모든 스타일이 CSS 변수를 참조하여 테마 전환 자동 대응

## 테마 시스템

React Context를 통해 테마를 전체 앱에 전파한다:

- **ThemeProvider**: 앱 루트에 배치하여 Design Token 값을 CSS 변수로 주입
- **Light/Dark 모드**: CSS 변수 값만 교체하여 테마 전환
- **커스텀 테마**: 기업/고객별 브랜드 색상을 토큰 오버라이드로 적용 가능

## 배포 및 버전 관리

| 항목 | 전략 |
|------|------|
| 배포 방식 | 사내 npm 레지스트리 또는 GitHub Packages |
| 버전 정책 | Semantic Versioning (semver) |
| 변경 로그 | Conventional Commits 기반 자동 생성 |
| 시각적 테스트 | Chromatic(Storybook) 기반 시각적 회귀 테스트 |
| 문서화 | Storybook으로 인터랙티브 문서 제공 |

### Module Federation과의 통합

모든 Remote App이 React를 사용하므로, `@portal/react-ui`를 Module Federation의 공유 디펜던시로 설정할 수 있다. 이 경우 컴포넌트 라이브러리가 Shell에서 한 번만 로딩되고 모든 앱이 공유하여 번들 크기를 추가로 절감한다.

## 이기종 프레임워크 대응

전체 앱이 React로 통일되지 않은 경우의 대응 전략:

| 시나리오 | 대응 |
|---------|------|
| 소수의 Vue/Angular 앱이 추가될 때 | 해당 앱은 Design Token(Tier 1)만 사용하고, Tier 2는 자체 구현 |
| Vue 앱이 다수가 될 때 | `@portal/vue-ui`를 별도로 구축 (Design Token 공유) |
| 프레임워크 불문 표준이 필요할 때 | Web Components fallback 레이어 추가 |

## 제약 사항 및 고려사항

1. **프레임워크 종속**: Vue나 Angular 앱에서는 사용할 수 없으므로, 이기종 앱 통합 시 별도 전략 필요
2. **React 버전 동기화**: 모든 Remote App이 동일한 React 메이저 버전을 사용해야 컴포넌트가 정상 동작
3. **라이브러리 유지보수**: 전담 팀 또는 오너가 있어야 장기적으로 유지 가능
4. **초기 투자**: 기초 컴포넌트부터 고급 데이터 그리드까지 구축하려면 상당한 초기 리소스 필요
