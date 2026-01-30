---
id: sec-iframe-sandbox
category: security-isolation
label: iframe 전면 격리
summary: 모든 Remote App을 iframe sandbox로 격리하여 최고 수준의 보안을 적용한다.
pros:
  - 모든 앱이 독립 브라우징 컨텍스트에서 실행되어 완벽한 JS/CSS/DOM 격리
  - 한 앱의 XSS 취약점이 다른 앱에 절대 전파되지 않음
  - sandbox 속성으로 각 앱의 기능을 세밀하게 제한 가능
  - 외부/서드파티 앱을 안전하게 통합할 수 있음
cons:
  - 앱 간 통신이 postMessage로만 가능하여 지연과 복잡도 증가
  - 디펜던시 공유 불가로 각 앱이 모든 라이브러리를 개별 로딩 (번들 크기 증가)
  - iframe당 별도 브라우저 컨텍스트 → 메모리 사용량 대폭 증가
  - 반응형 레이아웃, 키보드 단축키, 접근성 등 UX 문제 다수
bestFor:
  - 모든 Remote App이 외부 벤더에서 개발된 멀티테넌트 플랫폼
  - 보안 규제가 극히 엄격한 금융·의료·정부 시스템
  - 코드 리뷰가 불가능한 서드파티 위젯을 다수 포함하는 포털
---

## 개요

iframe 전면 격리 전략은 모든 Remote App을 예외 없이 iframe으로 로딩하는 방식이다. 내부 앱과 외부 앱의 구분 없이 동일한 격리 수준을 적용하여, 보안 모델을 단순하고 일관되게 유지한다.

이 방식은 "기본적으로 불신(Default Deny)"이라는 보안 원칙을 철저히 따른다. 모든 앱은 격리된 환경에서 실행되며, 필요한 기능만 sandbox 속성으로 명시적으로 허용한다.

## 아키텍처

### 격리 구조

```
Shell App (Host Page)
├── Header / Sidebar / Tab Bar
└── Content Area
    ├── <iframe> Remote App A (내부 앱이지만 격리)
    │   └── 독립 window/document/JS heap
    ├── <iframe> Remote App B (내부 앱이지만 격리)
    │   └── 독립 window/document/JS heap
    └── <iframe> Remote App C (외부 앱, 격리)
        └── 독립 window/document/JS heap
```

### Sandbox 정책 등급

앱의 신뢰 수준과 필요 기능에 따라 sandbox 속성을 차등 적용한다:

| 등급 | sandbox 속성 | 대상 |
|------|-------------|------|
| **Level 1 (최소 권한)** | `allow-scripts` | 정적 콘텐츠 표시용 위젯 |
| **Level 2 (표준)** | `allow-scripts allow-same-origin allow-forms` | 일반 업무 앱 |
| **Level 3 (확장)** | `allow-scripts allow-same-origin allow-forms allow-popups` | 팝업/새 창이 필요한 앱 |
| **Level 4 (완전)** | sandbox 속성 없음 (iframe만 사용) | 동일 출처 내부 앱 (성능 우선) |

### 통합 통신 레이어

모든 앱이 iframe이므로, Shell과의 통신을 표준화한 프로토콜이 필수다.

**PortalMessage 프로토콜:** 모든 메시지는 통일된 형식을 따른다. source 필드는 메시지 출처를 식별하며 'shell' 또는 앱 ID 문자열이다. type 필드는 메시지 유형으로 AUTH_UPDATE, NAVIGATE, EVENT, RESIZE, READY, TITLE_CHANGE, DIRTY_STATE, REQUEST, RESPONSE 등이 있다. payload에 실제 데이터를 담고, 요청-응답 패턴을 위한 correlationId와 디버깅/로깅용 timestamp를 포함한다.

**ShellMessageRouter:** Shell 측의 메시지 라우터는 등록된 iframe들의 참조와 메시지 핸들러를 Map으로 관리한다. window의 message 이벤트를 구독하여 메시지 유효성 검사와 출처 허용 여부를 확인한 후 적절한 핸들러로 전달한다. sendTo 메서드로 특정 앱에게 메시지를 전송하고, broadcast 메서드로 모든 등록된 iframe에 메시지를 일괄 전송할 수 있다. 메시지 전송 시 source를 'shell'로, timestamp를 현재 시각으로 자동 설정한다.

## 인증 토큰 전달

iframe 격리 환경에서 인증 토큰을 안전하게 전달하는 방법:

| 방법 | 보안 수준 | 설명 |
|------|----------|------|
| **postMessage** | 높음 | Shell이 origin 검증 후 토큰 전달 |
| **쿠키 (SameSite)** | 중간 | 동일 도메인이면 쿠키 자동 전송 |
| **URL 파라미터** | 낮음 (비권장) | 히스토리에 토큰 노출 위험 |

권장 방식은 postMessage를 활용하는 것이다. Shell은 iframe의 load 이벤트가 완료된 후 AUTH_UPDATE 유형의 메시지로 액세스 토큰, 현재 사용자 정보, 권한 목록을 전달한다. 토큰이 갱신될 때에는 인증 서비스의 갱신 이벤트를 구독하여 모든 iframe에 새 토큰을 broadcast로 일괄 전파한다.

## 메모리 관리

iframe 전면 격리는 메모리 소비가 가장 큰 방식이므로, 적극적인 메모리 관리가 필요하다.

IframeLifecycleManager 클래스를 통해 동시 활성 iframe 수를 제한한다(예: 최대 5개). LRU(Least Recently Used) 큐를 유지하여 활성화 순서를 추적하고, iframe이 활성화될 때마다 큐를 갱신한다. 제한을 초과하면 가장 오래 사용되지 않은 iframe부터 제거한다. iframe 제거 시에는 src를 'about:blank'로 변경하여 리소스를 해제한 뒤 DOM에서 제거하고, 내부 참조 Map에서도 삭제한다.

## 성능 특성

| 항목 | 수치/특징 |
|------|----------|
| 초기 로딩 | 각 iframe이 독립 페이지를 로딩 → 앱당 1~3초 추가 |
| 앱 간 통신 | postMessage 직렬화/역직렬화 → 10~50ms |
| 메모리 | iframe당 50~100MB → 5개 앱 동시 실행 시 500MB+ |
| 번들 크기 | 디펜던시 공유 불가 → 전체 번들 크기 2~3배 증가 |

## 제약 사항 및 고려사항

1. **성능 오버헤드**: 내부 앱까지 iframe으로 격리하면 불필요한 성능 비용 발생
2. **UX 저하**: 모달, 드롭다운, 툴팁 등이 iframe 경계를 넘지 못함
3. **개발 경험**: postMessage 기반 통신은 타입 안전성이 낮고 디버깅이 어려움
4. **SEO/접근성**: iframe 내부 콘텐츠는 검색 엔진과 스크린 리더가 접근하기 어려움
5. **반응형**: iframe 높이 자동 조절이 까다로우며, 중첩 스크롤이 발생할 수 있음
