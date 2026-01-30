---
id: ds-vue-ui
category: design-system
label: Vue 네이티브 컴포넌트
summary: Design Token 기반 위에 Vue 네이티브 컴포넌트 라이브러리(@portal/vue-ui)를 구축하여 Vue 생태계에 최적화된 개발 경험을 제공한다.
pros:
  - Vue의 template 문법, v-model, slots, provide/inject와 완벽 통합
  - Vue DevTools와 연동되어 컴포넌트 상태 디버깅이 용이
  - Composition API와 Options API 모두 지원 가능
  - Nuxt.js 등 Vue 생태계 프레임워크와 자연스러운 연동
cons:
  - Vue가 아닌 앱(React, Angular 등)에서는 사용 불가
  - Vue 2와 Vue 3의 API 차이가 크므로, 둘 다 지원하려면 별도 빌드 필요
  - React 대비 컴포넌트 라이브러리 생태계(서드파티)가 적음
bestFor:
  - 모든 또는 대부분의 Remote App이 Vue로 개발되는 프로젝트
  - Vue 3 + Composition API를 표준으로 채택한 조직
  - Element Plus, Vuetify 등 Vue 기반 UI 프레임워크를 기반으로 확장하려는 경우
---

## 개요

Vue 네이티브 컴포넌트 전략은 Design Token(Tier 1) 위에 Vue 전용 UI 컴포넌트 라이브러리(Tier 2)를 구축하는 방식이다. `@portal/vue-ui` 패키지로 배포되며, 모든 Vue 기반 Remote App이 이를 import하여 일관된 UI를 구현한다.

Vue의 양방향 바인딩(v-model), 슬롯(slots), provide/inject 등 Vue 고유의 패턴을 그대로 활용할 수 있어, Vue 개발자에게 가장 자연스러운 경험을 제공한다.

## 컴포넌트 라이브러리 구성

### 패키지 구조

```
@portal/design-tokens   → Tier 1: CSS 변수, JSON 토큰
@portal/vue-ui          → Tier 2: Vue 3 컴포넌트 라이브러리
@portal/vue-icons       → 아이콘셋 (Vue 컴포넌트 형태)
```

### 컴포넌트 분류

React 버전과 동일한 범위의 컴포넌트를 Vue 네이티브로 구현한다:

| 분류 | 컴포넌트 | Vue 고유 기능 활용 |
|------|---------|-------------------|
| **기초** | PButton, PInput, PSelect, PCheckbox 등 | v-model 양방향 바인딩 |
| **레이아웃** | PStack, PGrid, PFlex | scoped slots로 유연한 레이아웃 |
| **데이터 표시** | PTable, PDataGrid, PCard | 가상 스크롤, 정렬/필터 내장 |
| **피드백** | PToast, PModal, PDrawer | Teleport로 DOM 위치 제어 |
| **폼** | PForm, PFormField, PDatePicker | Vue 폼 검증 라이브러리와 통합 |

### 설계 원칙

1. **v-model 지원**: 모든 입력 컴포넌트는 v-model로 양방향 바인딩 가능
2. **Slots 기반 커스터마이징**: named slots와 scoped slots로 컴포넌트 내부를 유연하게 대체 가능
3. **접근성 내장**: WAI-ARIA, 키보드 내비게이션 기본 포함
4. **Design Token 연동**: CSS 변수를 통한 테마 자동 대응

## 테마 시스템

Vue의 provide/inject 패턴으로 테마를 전파한다. 앱 루트에서 ThemeProvider를 배치하면, 하위 모든 컴포넌트가 현재 테마(Light/Dark)와 토큰 값을 자동으로 참조한다. CSS 변수 기반이므로 React 버전과 동일한 토큰을 공유한다.

## 기존 Vue UI 프레임워크 활용

처음부터 모든 컴포넌트를 구축하는 대신, 기존 Vue UI 프레임워크를 기반으로 확장하는 전략도 가능하다:

| 기반 프레임워크 | 특징 | 적합 상황 |
|---------------|------|----------|
| **Element Plus** | 가장 풍부한 컴포넌트 셋, 엔터프라이즈 친화적 | ERP/MES 등 데이터 집약적 시스템 |
| **Vuetify 3** | Material Design 기반, 접근성 우수 | 일반 업무 포털 |
| **Naive UI** | TypeScript 기반, 트리 셰이킹 최적화 | 성능 중심 프로젝트 |
| **자체 구축** | 완전한 제어, 외부 의존성 없음 | 장기적 유지보수가 중요한 경우 |

기반 프레임워크를 사용할 경우, Design Token으로 테마를 오버라이드하고 포털 브랜드에 맞게 커스터마이징한다.

## 이기종 프레임워크 대응

| 시나리오 | 대응 |
|---------|------|
| 소수의 React 앱이 추가될 때 | 해당 앱은 Design Token만 사용 |
| React 앱이 다수가 될 때 | `@portal/react-ui`를 별도 구축 |
| 프레임워크 불문 표준 필요 시 | Web Components fallback 추가 |

## 제약 사항 및 고려사항

1. **Vue 버전 통일**: Vue 2와 Vue 3의 API가 크게 다르므로, 포털 전체에서 Vue 3를 표준으로 채택해야 함
2. **React 앱 미지원**: React 앱에서는 별도 대응이 필요하며, 이는 이중 유지보수 비용 발생
3. **생태계 규모**: React 대비 서드파티 컴포넌트 라이브러리와 도구가 적음
4. **SSR 고려**: Nuxt.js 기반 앱과 SPA 기반 앱의 컴포넌트 렌더링 방식 차이 관리 필요
