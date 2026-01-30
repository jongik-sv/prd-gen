# Part 2: 포털 설계 심화 토의 - 대화형 토론 기록

## 토의 참석자
| 분야 | 인물 |
|------|------|
| AI | Andrew Ng + Andrej Karpathy |
| DB | Martin Kleppmann |
| 아키텍처 | Martin Fowler |
| SW | Robert C. Martin (Uncle Bob) |
| PM | Kent Beck (사회자) |
| UX/보안 | Jakob Nielsen + Bruce Schneier |

---

## 토의 1: Micro Frontend 통합 방식 결정

**Beck (사회자):** 자, 포털 설계를 시작합시다. 첫 번째 질문입니다. **포털의 기술적 뼈대를 어떤 구조로 가져갈 건가요?** Fowler 먼저 시작하시죠.

---

**Fowler:** 선택지는 세 가지입니다.

1. **Module Federation** (Webpack 5 / Vite) - 런타임에 원격 모듈을 로딩
2. **Single-SPA** - 프레임워크 독립적 오케스트레이터
3. **iframe** - 완벽한 격리, 최악의 UX

저는 **Module Federation**을 추천합니다. 이유는 간단합니다. 공유 디펜던시 최적화가 됩니다. React를 쓰는 프로젝트 3개가 있으면, React를 한 번만 로딩합니다. Single-SPA는 이게 안 됩니다.

**Uncle Bob:** 잠깐요. 프로젝트마다 **다른 프레임워크**를 쓴다고 했습니다. React만 쓰는 게 아니라 Vue, Angular도 섞일 수 있다면, Module Federation의 공유 디펜던시 이점이 줄어듭니다. 그 경우 Single-SPA가 더 나을 수 있습니다.

**Fowler:** 좋은 지적입니다. 그렇다면 **현실적 조합**을 봅시다. 대부분의 프로젝트가 React 계열이고, 일부만 Vue/Angular라면 Module Federation이 유리합니다. 반대로 정말 제각각이면 Single-SPA가 맞습니다.

**Beck:** 이건 프로젝트 현황을 봐야 결정할 수 있는 문제네요. **결정을 미루는 게 아니라, 두 방식 모두 수용할 수 있는 Shell App을 설계하는 게 답**이 아닐까요?

**Fowler:** 맞습니다. Shell App의 계약(Contract)만 잘 정의하면, Module Federation이든 Single-SPA든 iframe이든 Remote App을 마운트하는 방식은 **어댑터 레이어**에서 바꿀 수 있습니다.

> **합의:** Shell App은 통합 방식에 무관한 계약 기반 설계. Adapter 패턴으로 Module Federation / Single-SPA / iframe 모두 수용 가능하게.

---

## 토의 2: Shell App 책임 범위 및 Contract 정의

**Beck:** 좋습니다. 그러면 Shell App이 **구체적으로 뭘 담당하는지** 명확히 합시다.

**Fowler:** Shell App의 책임은 정확히 5가지입니다:

```
1. 인증 상태 관리 - SSO 로그인/로그아웃, Token 보관, 갱신
2. 글로벌 레이아웃 - Header, Sidebar, Footer
3. 라우팅 - URL 경로 -> Remote App 매핑, 앱 전환
4. 이벤트 버스 - Shell <-> Remote App 간 통신 채널
5. 글로벌 기능 - 통합 검색, 알림, 사용자 프로필
```

그 이상은 Shell이 담당하면 안 됩니다. Shell이 비대해지면 각 프로젝트의 독립성이 무너집니다.

**Uncle Bob:** 이 5가지를 **인터페이스로 명확히 정의**해야 합니다. 예를 들어:

```typescript
// Remote App이 구현해야 하는 계약
interface RemoteApp {
  // 앱 생명주기
  bootstrap(): Promise<void>;   // 최초 1회: 리소스 초기화
  mount(container: HTMLElement, context: AppContext): Promise<void>;   // DOM에 마운트
  unmount(): Promise<void>;     // DOM에서 제거

  // 메타데이터
  manifest: {
    id: string;                 // 고유 식별자
    name: string;               // 표시명
    routes: RouteConfig[];      // 라우팅 경로 목록
    permissions: string[];      // 필요 권한 목록
    version: string;
  };
}

// Shell이 Remote App에 제공하는 컨텍스트
interface AppContext {
  auth: {
    user: UserInfo;
    token: string;
    hasPermission(permission: string): boolean;
  };
  navigation: {
    navigate(path: string): void;
    getCurrentPath(): string;
  };
  events: {
    emit(event: string, payload: any): void;
    on(event: string, handler: Function): Unsubscribe;
  };
  i18n: {
    locale: string;
    t(key: string): string;
  };
  masterData: {
    getCode(groupId: string, codeId: string): CodeInfo;
    getCodeList(groupId: string): CodeInfo[];
  };
}
```

**Karpathy:** 잠깐, `AppContext`에 `masterData`를 직접 넣는 건 좀 고민이 됩니다. Remote App이 마스터 데이터를 Shell을 통해 간접적으로 접근하면, **Shell이 Single Point of Failure**가 됩니다. 차라리 Master SDK를 Remote App에 직접 주입하는 게 낫지 않나요?

**Uncle Bob:** 좋은 지적입니다. `masterData`는 Shell Context에서 빼고, **별도 SDK import**로 가는 게 맞습니다. Shell Context에는 인증과 네비게이션만 넣고, 나머지는 독립 SDK로.

**Fowler:** 동의합니다. Shell Context는 **최소한**으로 유지해야 합니다. 원칙을 정리하면:

```
Shell Context에 넣을 것: auth, navigation, events (Shell 고유 기능)
별도 SDK로 제공할 것: masterData, analytics, feature-flags (독립 기능)
```

> **합의:** Shell Context는 auth, navigation, events만. masterData 등은 독립 SDK로 분리.

---

## 토의 3: UX 최적화 - 앱 전환 경험

**Nielsen:** 여기서 UX 관점을 넣겠습니다. **사용자가 앱을 전환할 때 어떤 경험을 하는지**가 중요합니다.

현재 대부분의 Micro Frontend 포털에서 가장 큰 불만은:
1. 앱 전환 시 **1-2초 빈 화면** (Remote App 로딩 시간)
2. **스크롤 위치 초기화** (이전 앱 상태 소실)
3. **스타일 깜빡임** (CSS 로딩 순서 차이)

이걸 해결하려면:

```
1. Skeleton UI: 앱 로딩 중 레이아웃 뼈대를 먼저 표시
2. 프리페칭: 사이드바에서 마우스 호버 시 다음 앱을 미리 로딩
3. 앱 상태 캐싱: unmount 시 상태를 SessionStorage에 보관, 재진입 시 복원
4. CSS-in-JS 또는 Scoped CSS: 앱 간 스타일 충돌 방지
```

**Fowler:** 프리페칭은 좋은 아이디어인데, **언제** 프리페치할지가 중요합니다. 모든 앱을 초기에 로딩하면 포털 자체가 느려집니다. 제안:

```
즉시 로딩: 현재 라우트의 앱
프리페치: 사이드바 상위 3개 메뉴의 앱
지연 로딩: 나머지 (사용자 접근 시)
```

**Karpathy:** 여기에 AI를 적용할 수 있습니다. 사용자의 **메뉴 접근 패턴을 학습**해서, 다음에 접근할 가능성이 높은 앱을 예측하고 프리페치하는 것입니다. 하지만 이건 Phase 4의 영역이고, Phase 1에서는 Nielsen이 말한 상위 3개 메뉴 방식이면 충분합니다.

> **합의:** Skeleton UI + 상위 메뉴 프리페칭 + 앱 상태 캐싱을 Phase 1에 포함.

---

## 토의 4: Remote App 간 보안 격리

**Schneier:** 보안 관점에서 한 가지 중요한 결정이 있습니다. **Remote App 간 격리 수준**입니다.

Module Federation은 같은 JavaScript 컨텍스트를 공유합니다. 즉, 악의적인 Remote App이 다른 앱의 전역 변수, DOM, 쿠키에 접근할 수 있습니다.

선택지:
```
1. 같은 컨텍스트 (Module Federation 기본): 성능 최고, 격리 없음
2. Shadow DOM: 스타일 격리, JS는 공유
3. Web Worker: JS 격리, DOM 접근 불가 (비현실적)
4. iframe: 완벽한 격리, UX 최악
```

**내부 프로젝트**만 올라가는 포털이라면, 1번(같은 컨텍스트) + CSP 헤더로 충분합니다. **외부 프로젝트(서드파티)**도 올라간다면, iframe을 fallback으로 두는 하이브리드가 필요합니다.

**Uncle Bob:** 현실적인 판단입니다. 내부 프로젝트는 코드 리뷰와 CI/CD 파이프라인으로 신뢰를 확보하고, 외부 프로젝트만 iframe 샌드박스를 적용하는 **2-tier 전략**이 맞습니다.

> **합의:** 내부 앱은 Module Federation(같은 컨텍스트), 외부/신뢰불가 앱은 iframe 격리. 2-tier 보안 모델.

---

## 토의 5: 라우팅 설계 & App Registry

**Beck:** 다음으로, **라우팅 설계**를 구체화합시다. 사용자가 `/project-a/dashboard`에 접근하면 어떤 일이 일어나는지, 시퀀스를 그려봅시다.

**Fowler:** 전체 흐름입니다:

```
사용자 -> /project-a/dashboard 접근
    |
    v
[1] Shell App 로딩 (이미 로딩됨)
    |
    v
[2] Shell Router: "/project-a/*" -> Remote App A 매핑 확인
    |
    v
[3] Auth Check: 현재 사용자가 project-a 접근 권한 있는지 확인
    |-- 권한 없음 -> 403 페이지 또는 권한 요청 안내
    |-- 미인증 -> SSO 로그인 리다이렉트
    |
    v
[4] Remote App A 로딩 (프리페치됨이면 즉시, 아니면 fetch)
    |
    v
[5] Shell이 Remote App A를 컨테이너 DOM에 mount
    |-- AppContext 전달 (auth, navigation, events)
    |-- 내부 경로 "/dashboard"를 Remote App A에 전달
    |
    v
[6] Remote App A가 자체 라우터로 "/dashboard" 렌더링
```

**Uncle Bob:** 여기서 중요한 건 **라우팅 등록 규약**입니다. 각 Remote App이 자기 경로를 어떻게 Shell에 등록하는지:

```json
// project-a/manifest.json
{
  "id": "project-a",
  "name": "프로젝트 A",
  "basePath": "/project-a",
  "entry": "https://cdn.example.com/project-a/remoteEntry.js",
  "routes": [
    { "path": "/dashboard", "name": "대시보드", "icon": "dashboard" },
    { "path": "/settings", "name": "설정", "icon": "settings" }
  ],
  "requiredPermissions": ["PROJECT_A_ACCESS"],
  "version": "1.2.0",
  "framework": "react",
  "loadStrategy": "module-federation"
}
```

**Nielsen:** `routes` 배열에 `icon`과 `name`이 있는 게 좋습니다. 이걸로 **사이드바 메뉴를 자동 생성**할 수 있습니다. 다만, 메뉴의 **표시 순서(order)**와 **그룹핑** 정보도 필요합니다.

```json
"routes": [
  {
    "path": "/dashboard",
    "name": "대시보드",
    "icon": "dashboard",
    "group": "main",
    "order": 1,
    "showInSidebar": true
  }
]
```

**Kleppmann:** 이 manifest를 **어디에 저장하고 어떻게 조회**하느냐가 문제입니다. 선택지:
1. **정적 설정**: Shell App 빌드 시 포함 (변경 시 Shell 재배포 필요)
2. **API 기반**: Registry Service에서 런타임에 조회 (동적, 하지만 장애 포인트 추가)
3. **하이브리드**: 정적 기본값 + API로 최신 버전 오버라이드

**Fowler:** 3번 하이브리드를 추천합니다. Shell 빌드 시 기본 manifest를 포함하되, 런타임에 Registry API를 호출해서 업데이트합니다. Registry가 죽어도 기본 manifest로 동작합니다.

> **합의:** App Registry Service 도입. Shell은 빌드 시 기본 manifest 포함 + 런타임에 Registry API로 최신 정보 갱신. 장애 시 fallback.

---

## 토의 6: Design System 전략

**Beck:** Design System 이야기를 안 할 수 없겠죠. Nielsen, 리드해주세요.

**Nielsen:** 기술 스택이 다른 앱들이 하나의 포털에서 **같은 모양**이어야 합니다. 이걸 달성하는 방법은 **Design Token**입니다.

```
Design Token이란?
- 색상, 타이포, 간격, 그림자, 모서리 등 시각적 속성을 변수로 정의
- 프레임워크에 독립적 (CSS 변수, JSON, SCSS 등으로 변환 가능)
```

구체적으로:

```css
/* tokens.css - 모든 앱이 이 파일을 import */
:root {
  /* Color */
  --color-primary: #1a73e8;
  --color-primary-hover: #1557b0;
  --color-error: #d93025;
  --color-success: #188038;
  --color-bg-primary: #ffffff;
  --color-bg-secondary: #f8f9fa;
  --color-text-primary: #202124;
  --color-text-secondary: #5f6368;

  /* Typography */
  --font-family: 'Pretendard', -apple-system, sans-serif;
  --font-size-xs: 12px;
  --font-size-sm: 14px;
  --font-size-md: 16px;
  --font-size-lg: 20px;
  --font-size-xl: 24px;

  /* Spacing */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 32px;

  /* Border Radius */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;

  /* Shadow */
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.1);
  --shadow-md: 0 4px 6px rgba(0,0,0,0.1);
  --shadow-lg: 0 10px 15px rgba(0,0,0,0.1);
}
```

**Uncle Bob:** Token만으로는 부족합니다. 버튼, 입력, 테이블, 모달 등 **기본 UI 컴포넌트**도 공통으로 제공해야 합니다. 하지만 React 컴포넌트를 Vue 프로젝트에서 못 쓰니까...

**Nielsen:** 그래서 **2-tier 전략**입니다:
```
Tier 1: Design Token (CSS 변수) - 모든 프레임워크에서 사용 가능
Tier 2: 프레임워크별 컴포넌트 라이브러리
        - @portal/react-ui (React 프로젝트용)
        - @portal/vue-ui (Vue 프로젝트용)
        - Web Components (프레임워크 불문 fallback)
```

**Karpathy:** Web Components를 Tier 2의 기본으로 가는 건 어떤가요? 한 번만 만들면 어디서든 씁니다.

**Nielsen:** 이론적으로는 맞지만, Web Components의 **DX(개발자 경험)**가 React/Vue 네이티브 컴포넌트보다 열등합니다. 타입 추론, 이벤트 핸들링, 상태 바인딩이 자연스럽지 않습니다. 현실적으로는 **가장 많이 쓰는 프레임워크용 네이티브 라이브러리 + Web Components fallback** 조합이 최선입니다.

> **합의:** Design Token(CSS 변수) + 주력 프레임워크 네이티브 컴포넌트 라이브러리 + Web Components fallback.

---

## 토의 7: AI-Executable 문서 수준 정의

**Beck:** 마지막으로 중요한 것. 이 모든 설계가 **Claude Code나 Gemini가 바로 구현할 수 있는 수준**이 되려면, 추가로 무엇이 필요합니까?

**Uncle Bob:** 명확합니다. AI가 코드를 생성하려면 다음이 필요합니다:

```
1. 디렉토리 구조 명세: 어떤 파일이 어디에 있어야 하는지
2. 인터페이스/타입 정의: TypeScript 수준의 구체적인 타입
3. API 엔드포인트 목록: 입력/출력 스키마 포함
4. 시퀀스 다이어그램: 주요 흐름의 단계별 호출 순서
5. 데이터 모델: ERD 또는 SQL 스키마
6. 환경 설정: 의존성, 환경 변수, 설정 파일
```

**Ng:** 그리고 **검증 기준**도 필요합니다. AI가 만든 코드가 올바른지 어떻게 확인하는가. 즉, **테스트 시나리오**를 문서에 포함해야 합니다.

**Beck:** 정리하면, PRD/TRD에 이 수준까지 포함되어야 AI가 구현 가능합니다:
```
PRD: 기능 요구사항 + 사용자 시나리오 + 검증 기준
TRD: 디렉토리 구조 + 타입 정의 + API 스키마 + DB 스키마 + 시퀀스 + 환경 설정
```

> **최종 합의:** PRD/TRD를 AI-executable 수준으로 작성하려면, 타입 정의/API 스키마/DB 스키마/디렉토리 구조/테스트 시나리오가 필수.

---

## 토의 8: MES형 다중 화면 동시 작업 패턴

**Beck (사회자):** 중요한 주제가 빠졌습니다. MES(Manufacturing Execution System)처럼 **화면이 수십~수백 개인 시스템**에서, 사용자가 여러 화면을 동시에 열어놓고 탭을 오가며 작업하는 패턴입니다. 생산 현장에서는 모니터링 화면 보면서 동시에 작업 지시서 입력하고, 품질 검사 결과도 확인해야 합니다. 이걸 포털에서 어떻게 지원할 겁니까?

---

**Nielsen:** 이건 정말 중요합니다. MES뿐 아니라 ERP, WMS, SCM 등 **산업용 시스템 전반의 공통 패턴**입니다. 사용자 행태를 분석하면:

```
MES 오퍼레이터의 전형적인 작업 패턴:
1. 생산 현황 대시보드 - 항상 열어둠 (실시간 모니터링)
2. 작업 지시서 - 수시로 전환하며 상태 업데이트
3. 자재 투입 화면 - 생산 시 입력
4. 품질 검사 - 공정 완료 시 확인
5. 설비 모니터링 - 이상 발생 시 확인

→ 동시에 3~5개 화면을 열어두고, 수초 간격으로 탭 전환
→ 화면 전환할 때 이전 화면의 입력 상태가 사라지면 치명적
→ 특정 화면은 브라우저 탭이 아니라 "항상 보이는 패널"이어야 함
```

기존 웹 포털의 **단일 화면 라우팅 모델**은 이 시나리오에 완전히 부적합합니다. `/project-a/dashboard`에서 `/project-a/quality`로 이동하면, 대시보드가 unmount 되어 사라집니다. MES 사용자는 이걸 용납하지 않습니다.

---

**Fowler:** 맞습니다. 여기서 **라우팅 모델을 근본적으로 다시 생각**해야 합니다. 선택지는 세 가지입니다:

```
방식 1: 브라우저 탭 기반
 - 각 화면을 별도 브라우저 탭으로 열기
 - 장점: 구현 간단, OS 레벨 멀티태스킹 활용
 - 단점: 탭 간 상태 공유 어려움, 탭 수 폭발, 각 탭마다 Shell 중복 로딩

방식 2: 포털 내부 탭(In-App Tab)
 - 포털 내에 자체 탭 UI 구현, 각 탭이 독립된 Remote App 인스턴스
 - 장점: 상태 공유 쉬움, Shell 1회 로딩, 통합된 UX
 - 단점: 메모리 관리 필요, 구현 복잡도 높음

방식 3: 하이브리드 워크스페이스
 - 포털 내부 탭 + 드래그로 탭을 분리/병합 (VS Code, Chrome DevTools 패턴)
 - 장점: 최고의 유연성, 멀티모니터 지원
 - 단점: 구현 난이도 최고
```

**Beck:** 각 방식의 트레이드오프가 명확하네요. 어떤 걸 택합니까?

---

**Nielsen:** 산업 현장의 현실을 봐야 합니다. 대부분의 MES 사용자는 **듀얼 모니터** 이상을 사용합니다. 한 모니터에 생산 현황, 다른 모니터에 작업 입력 화면을 띄우는 게 일반적입니다.

따라서 **방식 2(포털 내부 탭)를 기본**으로 하되, **탭을 별도 브라우저 창으로 분리하는 "Pop-out" 기능**을 추가하는 게 현실적입니다. 이게 방식 3의 간소화 버전입니다.

```
기본 모드: 포털 내부 탭 (같은 브라우저 창)
 └─ 탭 우클릭 → "새 창에서 열기" (Pop-out)
       └─ 별도 브라우저 창으로 분리
       └─ SharedWorker 또는 BroadcastChannel로 상태 동기화
```

**Uncle Bob:** Pop-out 개념은 좋은데, **구현 아키텍처**를 명확히 해야 합니다. Pop-out된 창은 Shell App의 인스턴스를 새로 로딩하는 겁니까, 아니면 경량 쉘만 로딩하는 겁니까?

**Fowler:** **경량 쉘(Lightweight Shell)**이어야 합니다. Pop-out 창에서 사이드바나 전체 메뉴가 필요 없습니다. 인증 상태만 공유하고, 해당 Remote App만 렌더링하면 됩니다.

```typescript
// Pop-out 창의 경량 쉘 구조
interface PopoutShell {
  // 메인 Shell과 인증 상태 공유 (BroadcastChannel)
  auth: SharedAuthState;

  // 최소한의 헤더 (앱 이름 + 메인으로 돌아가기 버튼)
  header: MinimalHeader;

  // 단일 Remote App 마운트
  mountApp(appId: string, route: string): void;

  // 메인 Shell과 이벤트 동기화
  eventBridge: BroadcastChannel;
}
```

---

**Kleppmann:** 여기서 **상태 동기화 문제**가 핵심입니다. 메인 창에서 작업 지시서의 상태를 "진행 중"으로 변경하면, Pop-out된 생산 현황 대시보드에도 **실시간으로 반영**되어야 합니다.

동기화 전략:

```
1. BroadcastChannel API (같은 Origin 내 탭/창 간 통신)
   - 장점: 브라우저 네이티브, 추가 서버 불필요
   - 단점: 같은 브라우저에서만 동작

2. WebSocket (서버 경유)
   - 장점: 다른 기기에서도 동기화 가능
   - 단점: 서버 부하

3. 하이브리드
   - 같은 브라우저 내: BroadcastChannel (즉시)
   - 다른 기기/브라우저: WebSocket (약간의 지연)
```

**Fowler:** 하이브리드가 맞습니다. 그리고 이 동기화 레이어는 **Shell의 EventBus를 확장**하는 형태가 되어야 합니다:

```typescript
interface PortalEventBus {
  // 로컬 이벤트 (현재 탭/창 내)
  emit(event: string, payload: any): void;
  on(event: string, handler: Function): Unsubscribe;

  // 크로스-윈도우 이벤트 (같은 브라우저의 다른 창/탭)
  broadcast(event: string, payload: any): void;
  onBroadcast(event: string, handler: Function): Unsubscribe;

  // 크로스-세션 이벤트 (서버 경유, 다른 기기)
  publish(channel: string, event: string, payload: any): void;
  subscribe(channel: string, handler: Function): Unsubscribe;
}
```

---

**Karpathy:** 한 가지 추가합니다. MES 환경에서는 **알림의 우선순위**가 중요합니다. 설비 이상 알림은 사용자가 어떤 화면에 있든 즉시 보여야 합니다. 일반 알림과 다르게, **긴급 알림은 모든 창에 동시에 오버레이로 표시**해야 합니다.

```
알림 우선순위 모델:
  P0 (Critical): 모든 창에 풀스크린 오버레이 + 사운드 + 진동
      예: 설비 비상 정지, 안전 사고
  P1 (High): 현재 포커스 창에 토스트 + 배지 + 사운드
      예: 품질 이탈, 자재 부족
  P2 (Normal): 알림 센터에 쌓임 + 배지
      예: 작업 완료, 승인 요청
  P3 (Low): 알림 센터에만 쌓임
      예: 공지사항, 시스템 업데이트
```

**Schneier:** 보안 관점에서, P0 알림은 **사용자가 무시할 수 없어야** 합니다. 생산 현장의 안전 이슈는 "닫기" 버튼으로 무시 가능해선 안 됩니다. 반드시 **확인(Acknowledge) 행위가 기록**되어야 합니다.

---

**Beck:** 좋습니다. 이제 **In-App 탭 시스템의 구체적인 동작 설계**를 합시다.

**Nielsen:** 핵심 UX 요구사항을 정리하겠습니다:

```
In-App 탭 시스템 UX 요구사항:

1. 탭 생성
   - 사이드바 메뉴 클릭 → 새 탭으로 열림 (기존 탭 유지)
   - Ctrl+Click → 강제 새 탭 (같은 화면을 2개 열 수 있음)
   - 최대 탭 수 제한: 기본 10개 (설정 가능)
   - 제한 초과 시: 가장 오래된 비활성 탭 자동 닫기 제안

2. 탭 전환
   - 클릭으로 전환 (당연)
   - Ctrl+1~9: 탭 번호로 직접 이동
   - Ctrl+Tab / Ctrl+Shift+Tab: 다음/이전 탭
   - 전환 시 이전 탭은 unmount하지 않음 (숨기기만)

3. 탭 상태 보존
   - 폼 입력 중인 데이터 유지
   - 스크롤 위치 유지
   - 테이블 정렬/필터 상태 유지
   - 로딩된 데이터 캐시 유지

4. 탭 표시 정보
   - 앱 아이콘 + 화면 이름
   - 수정 중인 데이터가 있으면 "●" 표시 (변경 감지)
   - 실시간 화면은 "◉" 표시 (라이브 상태)
   - 알림 배지 숫자

5. 탭 관리
   - 탭 드래그로 순서 변경
   - 탭 우클릭 컨텍스트 메뉴: 닫기, 다른 탭 모두 닫기, 새 창에서 열기
   - 닫기 시 미저장 데이터 있으면 확인 대화상자
```

---

**Uncle Bob:** 여기서 **아키텍처적으로 가장 어려운 부분**은 "탭 전환 시 unmount하지 않음"입니다. SPA에서 라우트가 바뀌면 기존 컴포넌트를 unmount하는 게 기본 동작인데, 이걸 무시해야 합니다.

구현 전략:

```typescript
// 탭 컨테이너 매니저
class TabContainerManager {
  private containers: Map<string, HTMLElement> = new Map();
  private apps: Map<string, MountedApp> = new Map();

  // 탭 활성화: 기존 앱은 숨기고, 대상 앱만 보이기
  activateTab(tabId: string): void {
    // 모든 컨테이너 숨기기
    this.containers.forEach((el, id) => {
      el.style.display = id === tabId ? 'block' : 'none';
    });

    // 비활성 탭의 requestAnimationFrame, setInterval 일시정지
    this.apps.forEach((app, id) => {
      if (id === tabId) {
        app.resume?.();   // 포커스 이벤트 발행
      } else {
        app.suspend?.();  // 백그라운드 전환 이벤트 발행
      }
    });
  }

  // 새 탭 열기: 컨테이너 생성 + 앱 마운트
  openTab(tabId: string, appId: string, route: string): void {
    const container = document.createElement('div');
    container.id = `tab-${tabId}`;
    this.rootElement.appendChild(container);
    this.containers.set(tabId, container);

    const app = this.loadApp(appId);
    app.mount(container, { ...this.context, initialRoute: route });
    this.apps.set(tabId, app);

    this.activateTab(tabId);
  }

  // 탭 닫기: 앱 unmount + 컨테이너 제거
  closeTab(tabId: string): void {
    const app = this.apps.get(tabId);
    app?.unmount();
    this.containers.get(tabId)?.remove();
    this.containers.delete(tabId);
    this.apps.delete(tabId);
  }
}
```

**Fowler:** 핵심은 `display: none` vs `unmount`의 차이입니다. `display: none`이면 DOM은 살아있고, React 컴포넌트도 마운트 상태를 유지합니다. 하지만 **메모리 문제**가 생깁니다. 10개 탭이 열려있으면 10개 앱이 모두 메모리에 있습니다.

---

**Kleppmann:** 메모리 관리는 정말 중요합니다. MES 오퍼레이터는 **8시간 교대 근무** 동안 브라우저를 닫지 않습니다. 탭을 계속 열고 닫으면 메모리 릭이 누적됩니다.

메모리 관리 전략:

```
1. LRU 기반 탭 Eviction
   - 최대 활성 탭 수: N개 (메모리 기반 동적 조절)
   - N 초과 시: 가장 오래 사용하지 않은 탭을 "휴면(suspend)" 처리
   - 휴면 탭: 앱 상태를 직렬화 → SessionStorage 저장 → unmount
   - 휴면 탭 재활성화: 상태 역직렬화 → 재마운트 (사용자에겐 투명)

2. 메모리 모니터링
   - performance.memory API로 힙 사용량 추적
   - 임계값(예: 1.5GB) 초과 시 자동 휴면 트리거
   - 사용자에게 "메모리 부족: 오래된 탭 정리 권장" 알림

3. 메모리 릭 방지 계약
   - Remote App 계약에 cleanup 메서드 추가
   - unmount 시 타이머, 이벤트 리스너, WebSocket 연결 반드시 해제
   - Shell이 unmount 후 잔존 리소스 감지 → 경고 로그
```

```typescript
// Remote App 생명주기 확장
interface RemoteApp {
  bootstrap(): Promise<void>;
  mount(container: HTMLElement, context: AppContext): Promise<void>;
  unmount(): Promise<void>;

  // 다중 탭 지원 추가 생명주기
  suspend?(): Promise<SerializedState>;  // 휴면: 상태 반환 후 정리
  resume?(state: SerializedState): Promise<void>;  // 복원: 저장된 상태로 재개

  // 리소스 사용량 보고 (Shell의 메모리 관리에 활용)
  getResourceUsage?(): { heapSize: number; domNodes: number; timers: number };
}
```

---

**Ng:** 여기에 한 가지 더 추가합니다. MES에서는 **같은 화면을 다른 파라미터로 여러 개 열어야** 하는 경우가 흔합니다. 예를 들어:

```
- 라인 1 생산 현황 (탭 1)
- 라인 2 생산 현황 (탭 2)  ← 같은 화면, 다른 라인
- 라인 3 생산 현황 (탭 3)  ← 같은 화면, 또 다른 라인
- 품질 검사 입력 (탭 4)    ← 다른 화면
```

이 경우 탭 식별이 중요합니다. 탭 제목이 모두 "생산 현황"이면 구분이 안 됩니다.

**Nielsen:** 맞습니다. **동적 탭 제목** 규칙이 필요합니다:

```
탭 제목 규칙:
  기본: "{화면명}"
  동일 화면 2개 이상 열릴 때: "{화면명} - {구분 파라미터}"

  예시:
  - "생산 현황 - 라인 1"
  - "생산 현황 - 라인 2"
  - "품질 검사 - LOT-2024-001"
```

이를 위해 Remote App은 **탭 제목을 동적으로 Shell에 알려주는** API가 필요합니다:

```typescript
// AppContext에 추가
interface AppContext {
  // ... 기존 항목 ...
  tab: {
    setTitle(title: string): void;           // 탭 제목 변경
    setBadge(count: number): void;           // 알림 배지
    setDirty(isDirty: boolean): void;        // 미저장 변경 표시
    setLive(isLive: boolean): void;          // 실시간 상태 표시
    requestClose(): void;                     // 탭 닫기 요청
    onCloseRequested(handler: () => boolean): void; // 닫기 전 확인
  };
}
```

---

**Beck:** 이제 **워크스페이스(Workspace)** 개념도 논의합시다. MES 사용자는 근무 시작 시마다 같은 탭 조합을 여는 게 번거로울 겁니다.

**Nielsen:** 완전히 맞습니다. **워크스페이스 프리셋** 기능이 필요합니다:

```
워크스페이스란?
  - 현재 열려있는 탭 목록 + 각 탭의 경로 + 레이아웃을 저장한 스냅샷
  - 사용자가 "근무 시작" 버튼 하나로 자기 역할에 맞는 화면들을 일괄 복원

예시 워크스페이스:
  "라인 1 오퍼레이터":
    - 탭 1: 생산 현황 (라인 1)
    - 탭 2: 작업 지시서 (라인 1)
    - 탭 3: 자재 투입
    - Pop-out 창: 설비 모니터링 (라인 1)

  "품질 관리자":
    - 탭 1: 품질 대시보드 (전체 라인)
    - 탭 2: SPC 차트
    - 탭 3: 검사 입력
    - 탭 4: 부적합 관리
```

```typescript
interface Workspace {
  id: string;
  name: string;
  owner: string;           // userId 또는 'system' (관리자 배포)
  isDefault: boolean;      // 로그인 시 자동 복원 여부
  tabs: WorkspaceTab[];
  popouts: WorkspacePopout[];
}

interface WorkspaceTab {
  appId: string;
  route: string;
  params: Record<string, string>;  // 라인 ID 등
  title: string;
  order: number;
}

interface WorkspacePopout {
  appId: string;
  route: string;
  params: Record<string, string>;
  windowPosition: { x: number; y: number; width: number; height: number };
}
```

**Uncle Bob:** 워크스페이스 저장은 **서버 측**이어야 합니다. 사용자가 다른 PC에서 로그인해도 자기 워크스페이스를 불러올 수 있어야 합니다. 그리고 관리자가 **역할별 기본 워크스페이스를 배포**할 수 있어야 합니다.

---

**Fowler:** 마지막으로 종합 아키텍처를 정리합시다. 기존 Shell App 설계에 다중 탭 지원을 반영하면:

```
Shell App 아키텍처 (다중 탭 포함)

┌──────────────────────────────────────────────┐
│  Shell App                                    │
│  ┌────────────────────────────────────────┐  │
│  │  Header  [통합검색] [알림🔔] [프로필]    │  │
│  ├────────────────────────────────────────┤  │
│  │  Tab Bar                               │  │
│  │  [생산현황-L1 ●] [작업지시 ●] [품질 ◉] [+]│ │
│  ├─────┬──────────────────────────────────┤  │
│  │     │  Tab Content Area                │  │
│  │  S  │  ┌──────────────────────────┐   │  │
│  │  i  │  │  [Active Tab: Remote App] │   │  │
│  │  d  │  │                           │   │  │
│  │  e  │  │  (mount된 앱 렌더링)       │   │  │
│  │  b  │  │                           │   │  │
│  │  a  │  └──────────────────────────┘   │  │
│  │  r  │  ┌──────────────────────────┐   │  │
│  │     │  │ [Hidden Tab: Remote App]  │   │  │
│  │  M  │  │ (display: none, 상태 유지) │   │  │
│  │  e  │  └──────────────────────────┘   │  │
│  │  n  │  ┌──────────────────────────┐   │  │
│  │  u  │  │ [Suspended Tab: 직렬화됨] │   │  │
│  │     │  │ (unmounted, 메모리 해제)   │   │  │
│  │     │  └──────────────────────────┘   │  │
│  └─────┴──────────────────────────────────┘  │
│  ┌────────────────────────────────────────┐  │
│  │  Status Bar  [메모리: 1.2GB] [탭: 5/10] │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘

   ↕ BroadcastChannel / EventBridge

┌──────────────────────┐
│  Pop-out Window       │
│  ┌──────────────────┐│
│  │ Minimal Header   ││
│  ├──────────────────┤│
│  │ Remote App       ││
│  │ (설비 모니터링)    ││
│  │                  ││
│  └──────────────────┘│
└──────────────────────┘
```

> **합의:**
> 1. **포털 내부 탭(In-App Tab)** 시스템을 Shell App의 핵심 기능으로 구현
> 2. **Pop-out** 기능으로 탭을 별도 브라우저 창으로 분리, 경량 쉘(Lightweight Shell)로 렌더링
> 3. **BroadcastChannel + WebSocket 하이브리드**로 창 간 상태 동기화
> 4. **LRU 기반 탭 Eviction**으로 메모리 관리 (활성/숨김/휴면 3단계)
> 5. **Remote App 생명주기 확장**: suspend/resume 메서드 추가
> 6. **동적 탭 제목 + 상태 표시**: dirty(●), live(◉), badge(숫자)
> 7. **워크스페이스 프리셋**: 탭 조합을 저장/복원, 역할별 기본값 관리자 배포
> 8. **우선순위 알림 모델**: P0~P3, P0은 모든 창에 강제 오버레이

---

## 토의 9: 다중 탭 환경에서의 라우팅 재설계

**Beck:** 기존 토의 5에서 URL 라우팅을 `/project-a/dashboard`로 정의했는데, 다중 탭이면 **URL 하나로 여러 탭을 표현할 수 없습니다**. 이걸 어떻게 해결합니까?

**Fowler:** 좋은 지적입니다. 두 가지 접근이 있습니다:

```
접근 1: URL은 활성 탭만 반영
  - URL: /portal/project-a/dashboard
  - 탭 전환 시 URL 갱신
  - 나머지 탭 정보는 SessionStorage에 저장
  - 장점: URL 단순, 북마크 가능
  - 단점: URL로 전체 탭 상태 복원 불가

접근 2: URL에 탭 상태 인코딩
  - URL: /portal?tabs=project-a:dashboard,project-b:quality&active=0
  - 장점: URL 공유로 탭 구성 복원 가능
  - 단점: URL 복잡, 길이 제한
```

**Uncle Bob:** 접근 1이 현실적입니다. URL은 **현재 보고 있는 것**만 반영하면 됩니다. 전체 탭 상태 복원은 **워크스페이스 기능**이 담당합니다. URL의 역할과 워크스페이스의 역할을 분리하는 겁니다.

```
URL의 역할: "지금 보고 있는 화면" + 공유/북마크
워크스페이스의 역할: "열려있는 전체 탭 + 레이아웃" 저장/복원
```

**Nielsen:** 동의합니다. 그리고 한 가지 더, 사용자가 URL을 직접 입력하거나 북마크로 들어올 때의 동작을 정의해야 합니다:

```
URL로 직접 접근 시 동작:
  1. 이미 해당 화면이 탭으로 열려있으면 → 해당 탭 활성화
  2. 열려있지 않으면 → 새 탭으로 열기
  3. 최초 접근(탭 없음)이면 → 기본 워크스페이스 복원 + 해당 탭 활성화
```

> **합의:** URL은 활성 탭만 반영. 전체 탭 상태는 워크스페이스로 관리. 직접 URL 접근 시 기존 탭 재사용 또는 신규 탭 생성.

---

## 토의 10: MDI(Multiple Document Interface) 체계 — 탭을 넘어선 레이아웃 전략

**Beck (사회자):** 토의 8에서 In-App 탭을 결정했는데, 한 단계 더 들어갑시다. 전통적인 MES/ERP 데스크톱 클라이언트들은 **MDI(Multiple Document Interface)** 체계를 사용했습니다. 하나의 부모 창 안에 여러 자식 창이 자유롭게 배치되는 구조입니다. 웹 포털에서 이걸 어디까지 구현할 건지, 그리고 구현해야 하는지부터 논의합시다.

---

**Nielsen:** 먼저 역사적 맥락을 짚겠습니다. 인터페이스 패러다임은 이렇게 진화했습니다:

```
MDI 진화 역사:

1. SDI (Single Document Interface)
   - 하나의 창 = 하나의 문서
   - 예: 메모장
   - 문제: 여러 문서 작업 시 창 관리 지옥

2. MDI (Multiple Document Interface) — 1990~2000년대 주류
   - 부모 창 안에 여러 자식 창 (cascade, tile, minimize 가능)
   - 예: Excel 97의 시트별 창, Visual Studio 6, 대부분의 MES/ERP 클라이언트
   - 장점: 단일 앱 내에서 여러 문서 동시 작업
   - 문제: 자식 창끼리 겹침, 위치 관리 혼란, 접근성 문제

3. TDI (Tabbed Document Interface) — 2000~2010년대
   - MDI의 자식 창을 탭으로 대체
   - 예: 브라우저, VS Code, 현대 IDE
   - 장점: 깔끔한 UI, 탭 전환이 직관적
   - 문제: 동시에 2개 이상 보려면 "분할(split)" 기능 필요

4. Docking/Panel Layout — 현재 주류
   - TDI + 자유로운 패널 분할/도킹
   - 예: VS Code (에디터 + 터미널 + 탐색기), Chrome DevTools, Bloomberg Terminal
   - 장점: 사용자가 자기 작업에 맞게 레이아웃을 자유롭게 구성
   - 문제: 구현 복잡도 매우 높음
```

MES 사용자가 실제로 원하는 건 **순수 MDI가 아닙니다**. 옛날 MDI의 "창 겹침"은 오히려 불편합니다. 원하는 건 **동시에 여러 화면을 한 눈에 보는 것**입니다. 즉, **Docking/Panel Layout에 가까운 것**을 원합니다.

---

**Fowler:** 웹에서 이걸 구현하는 방법을 구체적으로 봅시다. 선택지를 비교합니다:

```
레이아웃 전략 비교:

전략 A: 순수 TDI (탭만)
  ┌──────────────────────────────────┐
  │ [탭1] [탭2] [탭3]               │
  ├──────────────────────────────────┤
  │                                  │
  │    하나의 활성 화면만 표시         │
  │                                  │
  └──────────────────────────────────┘
  - 구현 난이도: ★☆☆☆☆
  - 동시 가시성: 1개
  - 적합: 일반 웹앱, 간단한 업무

전략 B: TDI + 분할 뷰 (Split View)
  ┌──────────────────────────────────┐
  │ [탭1] [탭2] | [탭3] [탭4]       │
  ├────────────────┬─────────────────┤
  │                │                 │
  │   좌측 패널     │   우측 패널     │
  │   (탭1 활성)    │   (탭3 활성)    │
  │                │                 │
  └────────────────┴─────────────────┘
  - 구현 난이도: ★★★☆☆
  - 동시 가시성: 2~4개 (2x2 그리드까지)
  - 적합: MES, ERP 등 산업용 시스템

전략 C: 자유 도킹 (Docking Layout)
  ┌────────────┬─────────────────────┐
  │            │  [탭2]              │
  │  [탭1]     ├──────────┬──────────┤
  │  (고정      │  [탭3]    │  [탭4]   │
  │   패널)     │          │          │
  │            │          │          │
  ├────────────┴──────────┴──────────┤
  │  [탭5] (하단 고정)                │
  └──────────────────────────────────┘
  - 구현 난이도: ★★★★★
  - 동시 가시성: 무제한 (사용자 자유 배치)
  - 적합: 트레이딩 터미널, IDE, 관제 시스템
```

**Beck:** 어떤 전략을 택합니까?

---

**Uncle Bob:** 여기서 **YAGNI(You Ain't Gonna Need It)** 원칙을 적용해야 합니다. 전략 C(자유 도킹)는 매력적이지만, 구현 비용이 엄청나고, 대부분의 MES 사용자는 **고정된 2~3개 분할**이면 충분합니다.

제 제안은 **전략 B(TDI + Split View)를 기본으로 구현**하되, **아키텍처는 전략 C로 확장 가능하게** 설계하는 겁니다.

**Fowler:** 동의합니다. 구체적으로:

```
Phase 1: TDI + Split View
  - 화면을 좌/우 또는 상/하로 분할 가능 (최대 2x2 = 4패널)
  - 각 패널이 독립된 탭 그룹을 가짐
  - 탭을 드래그해서 다른 패널로 이동 가능
  - 분할 비율 드래그로 조절 가능

Phase 2 (필요 시): Docking Layout
  - 자유로운 패널 분할/병합
  - 패널을 화면 가장자리에 도킹
  - 특정 화면을 "고정(pinned)" 패널로 설정
```

---

**Nielsen:** Split View의 **구체적인 UX 흐름**을 설계합시다:

```
Split View 조작 방법:

1. 분할 생성
   방법 A: 탭 우클릭 → "오른쪽에서 열기" / "아래에서 열기"
   방법 B: 탭을 드래그하여 화면 가장자리 스냅 영역에 놓기
   방법 C: 키보드 단축키 Ctrl+\ (좌우 분할), Ctrl+Shift+\ (상하 분할)

2. 분할 조절
   - 분할선(Divider) 드래그로 패널 크기 조절
   - 분할선 더블클릭: 50:50으로 리셋
   - 최소 패널 너비/높이: 300px (이하로 줄이면 자동 닫힘)

3. 분할 해제
   - 패널의 마지막 탭을 닫으면 패널 자동 제거
   - 패널의 모든 탭을 다른 패널로 드래그하면 제거

4. 드래그 스냅 가이드
   - 탭 드래그 시 화면에 반투명 가이드 표시 (VS Code 스타일)
   ┌──────────────────────────┐
   │         [상단]            │
   │   ┌──┬──────────┬──┐    │
   │   │좌│  [중앙]   │우│    │
   │   └──┴──────────┴──┘    │
   │         [하단]            │
   └──────────────────────────┘
   - 드래그를 가이드 영역에 놓으면 해당 방향으로 분할
```

---

**Karpathy:** 한 가지 중요한 시나리오가 있습니다. MES에서 **마스터-디테일(Master-Detail)** 패턴입니다. 왼쪽에 생산 주문 목록이 있고, 오른쪽에 선택된 주문의 상세 정보가 나오는 구조입니다. 이건 단순 Split View가 아니라, **두 패널 간의 데이터 연동**이 필요합니다.

```
마스터-디테일 패턴 예시:

┌───────────────────┬────────────────────────────┐
│  [생산 주문 목록]   │  [주문 상세: WO-2024-001]   │
│                   │                            │
│  WO-2024-001 ←──────── 클릭하면 우측 갱신        │
│  WO-2024-002     │  ┌─────────────────────┐   │
│  WO-2024-003     │  │ 품목: A제품          │   │
│                   │  │ 수량: 1,000         │   │
│                   │  │ 상태: 진행중         │   │
│                   │  └─────────────────────┘   │
└───────────────────┴────────────────────────────┘
```

이 경우 왼쪽 패널의 Remote App이 "주문 선택 이벤트"를 발행하고, 오른쪽 패널의 Remote App이 이를 수신해서 내용을 갱신해야 합니다.

**Fowler:** 이건 토의 2에서 정의한 **EventBus**로 해결됩니다. 하지만 패널 간 연동을 명시적으로 설정할 수 있는 메커니즘이 추가로 필요합니다:

```typescript
// 패널 연동(Linked Panels) 설정
interface PanelLink {
  id: string;
  sourcePanel: string;        // 마스터 패널 ID
  targetPanel: string;        // 디테일 패널 ID
  eventMapping: {
    sourceEvent: string;      // 예: "order:selected"
    targetAction: string;     // 예: "navigate"
    payloadTransform?: (payload: any) => any;  // 데이터 변환
  }[];
}

// 워크스페이스에 패널 링크 포함
interface Workspace {
  id: string;
  name: string;
  tabs: WorkspaceTab[];
  layout: PanelLayout;       // 분할 레이아웃 정보
  panelLinks: PanelLink[];   // 패널 간 연동 설정
}
```

---

**Uncle Bob:** 이제 **레이아웃 시스템의 데이터 구조**를 정의해야 합니다. Split View를 트리 구조로 표현합니다:

```typescript
// 레이아웃은 재귀적 트리 구조
type PanelLayout = PanelLeaf | PanelSplit;

interface PanelLeaf {
  type: 'leaf';
  panelId: string;
  tabs: TabState[];         // 이 패널에 속한 탭들
  activeTabIndex: number;
}

interface PanelSplit {
  type: 'split';
  direction: 'horizontal' | 'vertical';
  ratio: number;            // 0.0 ~ 1.0, 첫 번째 패널의 비율
  first: PanelLayout;       // 왼쪽/위
  second: PanelLayout;      // 오른쪽/아래
}

// 예시: 좌측 1개 + 우측 상하 2개 = 3패널
const layout: PanelLayout = {
  type: 'split',
  direction: 'horizontal',
  ratio: 0.4,
  first: {
    type: 'leaf',
    panelId: 'panel-1',
    tabs: [{ appId: 'mes', route: '/orders', title: '생산 주문 목록' }],
    activeTabIndex: 0
  },
  second: {
    type: 'split',
    direction: 'vertical',
    ratio: 0.6,
    first: {
      type: 'leaf',
      panelId: 'panel-2',
      tabs: [{ appId: 'mes', route: '/order-detail', title: '주문 상세' }],
      activeTabIndex: 0
    },
    second: {
      type: 'leaf',
      panelId: 'panel-3',
      tabs: [{ appId: 'mes', route: '/quality', title: '품질 검사' }],
      activeTabIndex: 0
    }
  }
};
```

**Kleppmann:** 이 트리 구조가 **워크스페이스 직렬화**의 핵심입니다. 이 JSON을 서버에 저장하면, 다른 기기에서 로그인해도 정확히 같은 레이아웃을 복원할 수 있습니다.

---

**Nielsen:** 여기서 한 가지 **중요한 UX 의사결정**이 있습니다. 사용자에게 레이아웃 자유도를 얼마나 줄 것인가입니다.

```
자유도 스펙트럼:

최소 자유도 ◄────────────────────────────► 최대 자유도
  고정 레이아웃    프리셋 선택      분할 지원      자유 도킹
  (관리자 지정)   (템플릿 기반)    (Split View)  (완전 자유)

추천: "프리셋 선택 + 분할 지원" 조합
  - 관리자가 역할별 레이아웃 프리셋을 배포 (기본값)
  - 사용자가 프리셋 위에서 추가로 분할/이동 가능
  - 변경된 레이아웃을 "내 레이아웃"으로 저장 가능
  - 프리셋으로 쉽게 초기화 가능 ("기본 레이아웃으로 복원")
```

이 접근의 장점은:
1. **초보 사용자**: 프리셋 그대로 사용 → 혼란 없음
2. **숙련 사용자**: 자기 스타일로 커스터마이즈 → 생산성 향상
3. **관리자**: 역할별 표준 화면 구성 통제 가능

---

**Schneier:** 보안 관점에서 한 가지. Split View에서 **서로 다른 프로젝트의 화면을 동시에 볼 때**, 권한 검증이 패널별로 독립적이어야 합니다. 왼쪽 패널에 프로젝트 A 권한이 있다고 해서, 오른쪽 패널에 프로젝트 B 화면이 자동으로 열려선 안 됩니다.

```
패널별 권한 검증 규칙:
  - 각 패널의 탭은 해당 Remote App의 권한을 독립 검증
  - 패널 링크(Master-Detail)에서 target 앱의 권한도 별도 확인
  - 권한 없는 앱을 패널에 드래그하면 → 403 안내 + 권한 요청 버튼
```

---

**Beck:** 좋습니다. 그러면 기존 레거시 MES에서 MDI를 사용하던 사용자를 위한 **마이그레이션 전략**도 논의합시다. 기존 C/S 클라이언트에서 웹 포털로 전환할 때, 사용자가 익숙한 작업 방식을 잃으면 안 됩니다.

**Nielsen:** 마이그레이션 관점에서 핵심 대응표입니다:

```
기존 MDI (C/S) → 웹 포털 대응

기존 MDI 기능              →  웹 포털 대응
──────────────────────────────────────────────────
자식 창 열기               →  새 탭 열기
자식 창 계단식 배치          →  탭 그룹 (지원 안 함, 불필요)
자식 창 타일 배치           →  Split View (2x2)
자식 창 최소화 → 하단 표시   →  탭으로 존재 (항상 접근 가능)
자식 창 드래그 이동          →  탭 드래그로 패널 간 이동
창 목록(Window List)       →  탭 바 + 전체 탭 목록 드롭다운
독립 창으로 분리            →  Pop-out (새 브라우저 창)
전체 화면 배치 저장          →  워크스페이스 저장
```

**사라지는 것**: MDI의 겹치는 자식 창, 자유 위치 지정
**대체되는 것**: Split View + 탭 + Pop-out 조합이 더 나은 경험 제공
**새로 생기는 것**: 워크스페이스 프리셋, 실시간 동기화, 크로스 디바이스 복원

---

**Ng:** 마지막으로 하나 더. **반응형(Responsive)** 고려입니다. MES는 주로 대형 모니터에서 사용하지만, 현장 태블릿에서 접근하는 경우도 있습니다.

```
디바이스별 레이아웃 적응:

데스크톱 (1920px+):
  - Split View 지원 (2x2 최대)
  - 사이드바 펼침
  - Pop-out 지원

태블릿 (768~1919px):
  - Split View 제한 (좌우 2분할만)
  - 사이드바 접힘 (햄버거 메뉴)
  - Pop-out 미지원 (브라우저 제한)

모바일 (< 768px):
  - 탭 전환만 (Split View 미지원)
  - 하단 탭 바로 전환
  - 핵심 기능만 표시 (간소화된 UI)
```

**Fowler:** 레이아웃 엔진이 **뷰포트 크기에 따라 자동 적응**해야 합니다. 태블릿에서 4분할 레이아웃을 열면, 자동으로 2분할로 축소되고, 나머지는 탭으로 전환됩니다.

```typescript
// 레이아웃 적응 규칙
interface LayoutPolicy {
  breakpoints: {
    desktop: { minWidth: 1920, maxPanels: 4, splitDirections: ['horizontal', 'vertical'] };
    laptop:  { minWidth: 1280, maxPanels: 3, splitDirections: ['horizontal', 'vertical'] };
    tablet:  { minWidth: 768,  maxPanels: 2, splitDirections: ['horizontal'] };
    mobile:  { minWidth: 0,    maxPanels: 1, splitDirections: [] };
  };

  // 패널 축소 시 동작
  onPanelOverflow: 'convert-to-tab' | 'hide' | 'stack';
}
```

> **합의:**
> 1. **TDI + Split View(전략 B)** 를 Phase 1으로 구현, 아키텍처는 전략 C(Docking) 확장 가능
> 2. **레이아웃 트리 구조**(PanelLayout)로 분할 상태를 재귀적 표현, 워크스페이스에 포함
> 3. **마스터-디테일 패턴**은 PanelLink로 패널 간 이벤트 연동
> 4. **프리셋 + 사용자 커스터마이즈** 조합으로 자유도와 통제의 균형
> 5. **반응형 레이아웃 적응**: 뷰포트 크기에 따라 패널 수 자동 조절
> 6. **MDI→웹 마이그레이션**: 기존 MDI 기능을 탭+Split View+Pop-out 조합으로 대체

---

## 포털 설계 핵심 결정 사항 요약

| # | 결정 사항 | 내용 |
|---|----------|------|
| 1 | Micro Frontend 방식 | Contract 기반 Shell + Adapter 패턴 (MF/Single-SPA/iframe 모두 수용) |
| 2 | Shell 책임 | auth, navigation, events만. 나머지는 독립 SDK |
| 3 | Remote App 계약 | bootstrap/mount/unmount 생명주기 + manifest |
| 4 | App Registry | 하이브리드 (빌드 시 기본값 + 런타임 API 갱신) |
| 5 | UX 최적화 | Skeleton UI + 상위 3메뉴 프리페칭 + 상태 캐싱 |
| 6 | 보안 격리 | 내부 앱: 같은 컨텍스트, 외부 앱: iframe (2-tier) |
| 7 | Design System | Tier 1(Design Token) + Tier 2(네이티브 컴포넌트 + Web Components) |
| 8 | 문서 수준 | AI-executable: 타입/API/DB스키마/디렉토리/테스트 포함 |
| 9 | 다중 탭 시스템 | In-App Tab + Pop-out + LRU 메모리 관리 + 워크스페이스 프리셋 |
| 10 | 라우팅 (다중 탭) | URL은 활성 탭만 반영. 전체 탭 상태는 워크스페이스로 관리 |
| 11 | MDI/레이아웃 | TDI + Split View (Phase 1) → Docking 확장 가능. 트리 구조 직렬화 |
| 12 | 패널 연동 | PanelLink로 마스터-디테일 패턴 지원 |
| 13 | 반응형 적응 | 뷰포트 크기별 최대 패널 수 자동 조절 |
