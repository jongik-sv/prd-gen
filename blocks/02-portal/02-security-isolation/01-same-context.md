---
id: sec-same-context
category: security-isolation
label: 동일 컨텍스트 (격리 없음)
summary: 모든 Remote App이 같은 JS/DOM 컨텍스트에서 실행되며, 코드 리뷰와 CI/CD 파이프라인으로 신뢰를 확보한다.
pros:
  - 앱 간 통신이 직접 함수 호출로 이루어져 지연 없이 가장 빠름
  - 디펜던시 공유가 자연스러워 번들 크기 최소화
  - 개발·디버깅이 단순하여 생산성이 높음
  - 앱 간 컴포넌트 공유, 상태 공유가 용이
cons:
  - 한 앱의 전역 변수 오염이 다른 앱에 직접 영향
  - 한 앱의 XSS 취약점이 전체 포털에 전파될 수 있음
  - CSS 충돌 가능성이 있어 네이밍 컨벤션 또는 CSS-in-JS 필수
  - 외부/서드파티 앱에는 적용 불가 (코드 리뷰가 안 되므로)
bestFor:
  - 모든 Remote App이 사내 팀에서 개발·관리하는 환경
  - 단일 프레임워크(React 등)로 통일된 프로젝트
  - 빠른 앱 간 통신이 필요한 실시간 시스템
---

## 개요

동일 컨텍스트 방식은 모든 Remote App이 Shell App과 같은 `window`, `document`, JavaScript 힙을 공유하는 가장 단순한 통합 방식이다. Module Federation이나 Single-SPA의 기본 동작이 이에 해당한다.

이 방식은 기술적 격리 장치를 두지 않는 대신, **조직적·프로세스적 보안**에 의존한다. 즉, 코드 리뷰, CI/CD 파이프라인의 보안 스캔, 린팅 규칙 등으로 악의적이거나 위험한 코드가 배포되지 않도록 방지한다.

## 보안 확보 전략

### 프로세스 기반 보안

기술적 격리가 없으므로, 다음의 프로세스적 장치가 필수다:

| 계층 | 장치 | 목적 |
|------|------|------|
| **코드 리뷰** | PR 리뷰에서 보안 체크리스트 적용 | 위험한 패턴 사전 차단 |
| **정적 분석** | ESLint 보안 규칙 (no-eval, no-innerHTML 등) | 자동화된 보안 검사 |
| **의존성 검사** | npm audit, Snyk, Dependabot | 취약한 패키지 탐지 |
| **CSP 헤더** | Content-Security-Policy 적용 | XSS 실행 차단 |
| **빌드 파이프라인** | CI에서 보안 스캔 통과 필수 | 배포 전 마지막 방어선 |

### CSP(Content Security Policy) 설정

CSP 헤더는 XSS 방어의 가장 효과적인 수단이다. 주요 정책은 다음과 같다. default-src를 'self'로 제한하여 기본적으로 동일 출처 리소스만 허용한다. script-src는 'self'와 CDN 도메인(예: cdn.example.com)만 허용하여 인라인 스크립트 실행을 차단한다. style-src는 'self'와 'unsafe-inline'을 허용하고, img-src는 'self', data URI, HTTPS를 허용한다. connect-src는 'self'와 API 서버 도메인 및 WebSocket 서버만 허용하며, frame-src는 'none'으로 설정하여 iframe 삽입을 차단한다.

### CSS 충돌 방지

동일 컨텍스트에서 여러 앱의 CSS가 공존하므로, 스타일 충돌을 방지해야 한다:

| 방법 | 설명 | 적용 난이도 |
|------|------|-----------|
| **CSS Modules** | 빌드 시 클래스명에 해시 접미사 추가 | 낮음 |
| **CSS-in-JS** | Styled Components, Emotion 등 런타임 스코핑 | 낮음 |
| **BEM 네이밍** | `.appA__header--active` 식의 네이밍 컨벤션 | 중간 (수동 관리) |
| **Shadow DOM** | Web Components로 스타일 캡슐화 | 높음 |

권장: CSS Modules 또는 CSS-in-JS를 포털 전체 표준으로 지정한다.

## 전역 변수 오염 방지

전역 변수 오염을 방지하기 위해 각 앱은 네임스페이스된 전역 객체만 사용해야 한다. window 객체에 __PORTAL_APPS__라는 공용 네임스페이스를 두고, 각 앱은 자신의 앱 ID를 키로 하여 해당 네임스페이스 하위에만 상태를 저장한다. window 객체에 직접 프로퍼티를 추가하는 것은 금지하며, ESLint 규칙으로 이를 자동 차단한다.

## 앱 간 오류 격리

기술적 격리가 없어도 React의 Error Boundary 패턴으로 앱 단위 오류 격리가 가능하다. Shell이 각 Remote App을 Error Boundary 컴포넌트로 감싸는 방식이다. Error Boundary는 getDerivedStateFromError로 렌더링 오류를 감지하고, componentDidCatch에서 에러 리포팅 서비스로 오류 정보(앱 ID, 에러 상세)를 전송한다. 오류 발생 시 해당 앱 영역에 대체 UI(fallback)를 표시하고 재시도 옵션을 제공한다.

이 방식은 렌더링 오류는 격리하지만, 비동기 오류(Promise rejection), 이벤트 핸들러 오류는 잡지 못한다. `window.onerror`와 `unhandledrejection` 이벤트도 함께 설정해야 한다.

## 성능 특성

| 항목 | 수치/특징 |
|------|----------|
| 앱 간 통신 | 직접 함수 호출, < 1ms |
| 메모리 | 디펜던시 공유로 앱당 추가 메모리 최소 |
| 앱 전환 | DOM show/hide로 즉각적 |
| 번들 크기 | 공유 디펜던시 1회 로딩 |

## 제약 사항 및 고려사항

1. **외부 앱 불가**: 코드 리뷰가 불가능한 외부/서드파티 앱은 이 방식으로 통합하면 안 됨
2. **팀 규모 제한**: 팀이 커지면 코드 리뷰만으로 모든 보안 위협을 차단하기 어려움
3. **책임 소재**: 한 앱의 버그가 다른 앱에 영향을 줄 때 원인 추적이 어려울 수 있음
4. **규제 환경**: 금융, 의료 등 규제 환경에서는 기술적 격리를 요구할 수 있음
