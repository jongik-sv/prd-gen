---
id: mf-iframe
category: mf-integration
label: iframe 완전 격리
summary: 각 Remote App을 iframe으로 로딩하여 완전한 브라우저 컨텍스트 격리를 제공한다.
pros:
  - 완벽한 JS/CSS/DOM 격리로 앱 간 간섭이 원천 차단됨
  - Remote App이 어떤 기술 스택이든 제한 없이 통합 가능
  - 보안 상 가장 안전하여 외부/서드파티 앱 통합에 적합
  - 한 앱의 크래시가 다른 앱에 전혀 영향을 주지 않음
cons:
  - 앱 간 통신이 postMessage로 제한되어 느리고 복잡함
  - 각 iframe이 별도의 브라우저 컨텍스트를 사용하므로 메모리 소비가 큼
  - 반응형 레이아웃 구현이 까다로움 (iframe 높이 자동 조절 문제)
  - SEO 불가, 접근성(a11y) 제약, 브라우저 히스토리 관리 어려움
  - 디펜던시 공유가 불가능하여 각 앱이 모든 라이브러리를 독립 로딩
bestFor:
  - 외부 벤더가 개발한 서드파티 앱을 안전하게 통합해야 하는 경우
  - 보안 격리가 최우선인 환경 (금융, 의료 등 규제 산업)
  - 레거시 시스템을 수정 없이 그대로 포함해야 하는 경우
---

## 개요

iframe은 웹 브라우저가 제공하는 가장 강력한 격리 메커니즘이다. 각 iframe은 독립된 브라우징 컨텍스트(separate browsing context)를 가지며, 자체 window, document, JavaScript 힙을 보유한다. 이로 인해 Remote App 간의 간섭이 원천적으로 불가능하다.

하지만 이 강력한 격리는 동시에 가장 큰 단점이기도 하다. Shell App과 Remote App 사이의 모든 통신이 `postMessage` API를 통해야 하며, DOM 접근이 교차 불가능하고, 반응형 레이아웃과 키보드/포커스 관리가 복잡해진다.

## 아키텍처

### iframe 통합 구조

```
Shell App (Host Page)
├── Header / Sidebar / Tab Bar
└── Content Area
    └── <iframe src="https://app-a.example.com/dashboard"
               sandbox="allow-scripts allow-same-origin allow-forms"
               style="width:100%; height:100%; border:none;">
        └── Remote App A (완전 독립 브라우징 컨텍스트)
            ├── 자체 React/Vue/Angular 런타임
            ├── 자체 CSS
            └── 자체 window/document
```

### Shell-iframe 통신 프로토콜

Shell과 iframe 간의 통신은 postMessage 기반의 구조화된 메시지 프로토콜을 따른다.

**Shell에서 iframe으로 보내는 메시지**는 type 필드로 메시지 유형을 구분한다. SHELL_CONTEXT_UPDATE는 인증 토큰과 사용자 정보를 전달하고, SHELL_NAVIGATE는 라우트 변경을 지시하며, SHELL_EVENT는 일반 이벤트를 전송한다. 요청-응답 패턴이 필요한 경우 requestId를 포함한다.

**iframe에서 Shell로 보내는 메시지**도 type 필드로 구분한다. APP_READY는 앱 초기화 완료를 알리고, APP_NAVIGATE는 Shell 라우터에 경로 반영을 요청하며, APP_RESIZE는 iframe 높이 자동 조절을 위해 콘텐츠 높이를 전달한다. APP_EVENT는 일반 이벤트 전송, APP_TITLE_CHANGE는 탭 제목 변경을 요청한다. 요청에 대한 응답 시에는 responseId를 포함한다.

### 통신 래퍼

Shell 측에서 iframe과의 통신을 관리하는 IframeBridge 클래스를 구현한다. 이 클래스는 iframe 요소에 대한 참조와 대기 중인 요청을 관리하는 Map을 보유한다. 생성 시 허용된 출처(allowedOrigin)를 설정하고, window의 message 이벤트를 구독하여 해당 출처에서 온 메시지만 처리한다. sendContext 메서드로 인증 상태를 iframe에 전달하고, request 메서드로 요청-응답 패턴의 비동기 통신을 수행한다. 요청 시 고유한 requestId를 생성하여 응답을 매칭한다.

## iframe 높이 자동 조절

iframe 통합에서 가장 빈번하게 발생하는 UX 문제는 iframe 높이가 내부 콘텐츠에 맞지 않는 것이다.

Remote App 측에서는 ResizeObserver를 사용하여 document.body의 크기 변화를 감지하고, 변경될 때마다 window.parent에 postMessage로 APP_RESIZE 유형의 메시지와 함께 새로운 높이 값을 전달한다.

Shell 측에서는 message 이벤트를 수신하여 APP_RESIZE 유형의 메시지가 도착하면, 해당 앱의 iframe 요소를 찾아 전달받은 높이 값으로 스타일을 갱신한다.

## Sandbox 속성

iframe의 `sandbox` 속성으로 세밀한 보안 정책을 적용할 수 있다:

| 속성 | 허용 내용 | 사용 여부 |
|------|----------|----------|
| `allow-scripts` | JavaScript 실행 | 필수 |
| `allow-same-origin` | 동일 출처 정책 적용 | 쿠키/스토리지 접근 필요 시 |
| `allow-forms` | 폼 제출 | 필요 시 |
| `allow-popups` | 팝업 창 열기 | Pop-out 지원 시 |
| `allow-modals` | alert/confirm/prompt | 비권장 (UX 차단) |
| `allow-top-navigation` | 상위 프레임 네비게이션 | 비허용 (보안 위험) |

권장 기본 설정은 sandbox 속성에 allow-scripts, allow-same-origin, allow-forms, allow-popups를 지정하는 것이다.

## 성능 특성

| 항목 | 수치/특징 |
|------|----------|
| 초기 로딩 | 각 iframe이 독립 페이지를 로딩하므로 가장 느림 (각 앱별 전체 번들 다운로드) |
| 앱 전환 | iframe 생성 + 페이지 로딩 → 1~3초 (캐시 미적용 시) |
| 메모리 | iframe당 별도 브라우징 컨텍스트 → 탭당 50~100MB 추가 |
| 통신 지연 | postMessage는 비동기 직렬화/역직렬화 → 복잡한 데이터 교환 시 지연 |

## 제약 사항 및 고려사항

1. **UX 한계**: iframe 내부의 모달/드롭다운이 iframe 경계를 넘을 수 없어, 잘림 현상 발생 가능
2. **키보드/포커스**: iframe과 호스트 간 포커스 전환이 자연스럽지 않으며, 키보드 단축키 충돌 관리 필요
3. **접근성(a11y)**: 스크린 리더가 iframe 경계를 넘나드는 것이 어려울 수 있음
4. **인쇄**: 브라우저 인쇄 시 iframe 내용이 잘리거나 누락될 수 있음
5. **딥링크**: iframe 내부의 URL 변경이 브라우저 주소창에 반영되지 않으므로, Shell이 별도로 라우트를 동기화해야 함
