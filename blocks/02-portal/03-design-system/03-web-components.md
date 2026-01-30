---
id: ds-web-components
category: design-system
label: Web Components
summary: 프레임워크에 독립적인 Web Components(Custom Elements + Shadow DOM)로 Design System을 구축하여 모든 프레임워크에서 사용 가능하게 한다.
pros:
  - 프레임워크에 완전히 독립적이어서 React, Vue, Angular, Svelte, Vanilla JS 어디서나 사용 가능
  - Shadow DOM으로 CSS가 자동 캡슐화되어 스타일 충돌 원천 차단
  - 웹 표준 기술이므로 특정 라이브러리에 종속되지 않아 장기적 안정성 확보
  - 한 번 구축하면 모든 프레임워크에서 그대로 사용 가능 (N개 프레임워크 × 1 구현)
cons:
  - React에서 Custom Element와의 호환이 완벽하지 않음 (이벤트 핸들링, props 전달 등)
  - 프레임워크 네이티브 컴포넌트 대비 DX(개발자 경험)가 열등 (타입 추론, 자동완성 부족)
  - Shadow DOM 내부 스타일링이 외부에서 어려워 커스터마이징에 제약
  - Server-Side Rendering(SSR) 지원이 제한적 (Declarative Shadow DOM은 아직 초기)
bestFor:
  - 3개 이상의 서로 다른 프레임워크가 혼재된 대규모 포털
  - 장기적으로 프레임워크 교체 가능성을 열어두고 싶은 경우
  - Design System을 외부 파트너에게도 배포해야 하는 플랫폼
---

## 개요

Web Components는 Custom Elements, Shadow DOM, HTML Templates라는 3가지 웹 표준 API의 조합이다. 이를 사용하면 `<portal-button>`, `<portal-table>` 같은 커스텀 HTML 요소를 정의하고, 어떤 프레임워크에서든 일반 HTML 태그처럼 사용할 수 있다.

이 방식의 핵심 가치는 **프레임워크 중립성**이다. React 앱에서든 Vue 앱에서든 동일한 `<portal-button>` 태그를 사용하며, Shadow DOM이 스타일을 자동으로 캡슐화하므로 CSS 충돌도 없다.

## 아키텍처

### 패키지 구조

```
@portal/design-tokens    → Tier 1: CSS 변수, JSON 토큰
@portal/web-components   → Tier 2: Web Components 라이브러리
@portal/react-wrappers   → (선택) React용 얇은 래퍼
@portal/vue-wrappers     → (선택) Vue용 얇은 래퍼
```

### 구현 기술 선택지

Web Components를 직접 구현하기보다, 개발 경험을 개선하는 라이브러리를 사용하는 것이 일반적이다:

| 라이브러리 | 특징 | 번들 크기 |
|-----------|------|----------|
| **Lit** | Google 개발, 가장 널리 사용, 경량 | ~5KB |
| **Stencil** | Ionic 팀 개발, TypeScript 기본, JSX 지원 | ~3KB (런타임) |
| **FAST** | Microsoft 개발, 고성능, Design Token 내장 | ~10KB |
| **Vanilla** | 순수 Custom Elements API | 0KB (추가 없음) |

Lit 또는 Stencil이 가장 성숙하고 커뮤니티 지원이 풍부하다.

## 프레임워크별 사용 경험

### React에서의 사용

React와 Web Components의 호환성은 React 19부터 크게 개선되었지만, 그 이전 버전에서는 다음 제약이 있다:

| 항목 | React 18 이하 | React 19+ |
|------|-------------|-----------|
| 문자열 속성 전달 | 정상 | 정상 |
| 객체/배열 props | ref로 수동 설정 필요 | 자동 전달 |
| 커스텀 이벤트 | addEventListener 수동 등록 | onXxx 패턴 사용 가능 |
| 타입 추론 | 수동 타입 정의 필요 | 개선됨 |

React 18 이하를 사용하는 프로젝트가 있다면, `@portal/react-wrappers`를 통해 React 친화적인 인터페이스를 제공하는 것이 권장된다.

### Vue에서의 사용

Vue는 Web Components와의 호환성이 우수하다. Vue의 템플릿 컴파일러가 Custom Element를 자동으로 인식하며, v-bind와 이벤트 바인딩이 자연스럽게 동작한다. 다만 v-model 자동 바인딩은 커스텀 이벤트 이름 컨벤션을 맞춰야 한다.

## Shadow DOM과 스타일링

Shadow DOM은 CSS 캡슐화를 제공하지만, 외부에서 내부 스타일을 변경하기 어렵다는 단점이 있다:

| 커스터마이징 방법 | 설명 | 유연성 |
|----------------|------|--------|
| **CSS 변수 (Design Token)** | Shadow DOM 내부에서 CSS 변수를 참조하면 외부에서 변수 값을 변경 가능 | 높음 |
| **::part() 의사 요소** | 컴포넌트가 part 속성을 노출하면 외부에서 해당 부분의 스타일을 지정 가능 | 중간 |
| **CSS Properties** | 컴포넌트가 커스텀 CSS 속성을 정의하여 외부에서 주입 가능 | 중간 |
| **Slot 기반** | 슬롯으로 내부 콘텐츠를 대체하여 커스터마이징 | 높음 |

Design Token(CSS 변수)을 적극 활용하면 Shadow DOM의 스타일링 제약을 대부분 해소할 수 있다.

## 성능 특성

| 항목 | 수치/특징 |
|------|----------|
| 번들 크기 | Lit 기반 컴포넌트는 개당 1~3KB, 프레임워크 런타임 불필요 |
| 렌더링 성능 | 네이티브 브라우저 API 사용으로 프레임워크 오버헤드 없음 |
| 초기 로딩 | 프레임워크 런타임이 없으므로 가장 가벼움 |
| 업데이트 성능 | Virtual DOM 없이 직접 DOM 조작 → 단순 UI에서 빠름, 복잡한 목록에서는 불리 |

## 제약 사항 및 고려사항

1. **DX 격차**: TypeScript 자동완성, JSX/템플릿 통합, DevTools 연동 등에서 네이티브 컴포넌트 대비 열등
2. **SSR 제한**: Shadow DOM의 서버 사이드 렌더링이 제한적이므로, SSR이 중요한 프로젝트에서는 별도 대응 필요
3. **폼 통합**: 브라우저 네이티브 폼 참여(form association)가 제한적이며, `ElementInternals` API 지원이 브라우저별로 상이
4. **테스트 도구**: React Testing Library, Vue Test Utils 등의 도구가 Web Components를 완벽하게 지원하지 않을 수 있음
5. **학습 곡선**: 팀이 Shadow DOM, Custom Elements 등 웹 표준 API에 익숙하지 않을 경우 학습 비용 발생
