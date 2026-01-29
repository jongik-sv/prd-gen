# Part 5: 통합 및 공통 설계 브레인스토밍

> **시리즈:** 7-Expert Brainstorming for PRD/TRD Design (Part 5 of 7)
> **주제:** 전체 시스템 통합 아키텍처, API Gateway/BFF, 공통 SDK, CI/CD, 관찰가능성, 테스트, DX
> **이전 파트:** Part 1 (아키텍처 개요), Part 2 (포털 설계), Part 3 (인증/인가 관리), Part 4 (마스터 데이터 관리)

## 참여 전문가

| # | 분야 | 인물 |
|---|------|------|
| 1 | AI | Andrew Ng + Andrej Karpathy |
| 2 | DB | Martin Kleppmann |
| 3 | 아키텍처 | Martin Fowler |
| 4 | SW | Robert C. Martin (Uncle Bob) |
| 5 | PM | Kent Beck (사회자) |
| 6 | UX/보안 | Jakob Nielsen + Bruce Schneier |

---

## 토의 5.1: 전체 시스템 아키텍처 다이어그램

**Beck:** 지금까지 Part 1에서 4까지 인증, 마스터 데이터, 포털 각각을 깊이 있게 논의했습니다. 이제 이 모든 것을 하나의 큰 그림으로 통합할 시간입니다. Fowler, 전체 아키텍처 다이어그램을 그려주시겠습니까?

**Fowler:** 네, 전체 시스템을 계층별로 정리하겠습니다. 아래 ASCII 다이어그램을 보시죠.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER (Public)                           │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                        Browser                                   │  │
│  │  ┌─────────────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │  │
│  │  │  Portal Shell    │  │Remote App│  │Remote App│  │Remote App│  │  │
│  │  │  (Host App)      │  │  Auth UI │  │  MDM UI  │  │ Proj App │  │  │
│  │  │  ┌────────────┐  │  └──────────┘  └──────────┘  └──────────┘  │  │
│  │  │  │ Portal SDK │  │                                            │  │
│  │  │  │ Auth SDK   │  │                                            │  │
│  │  │  │ Master SDK │  │                                            │  │
│  │  │  └────────────┘  │                                            │  │
│  │  └─────────────────┘                                             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        EDGE LAYER (Public)                             │
│  ┌──────────────┐    ┌──────────────────────────────────────────────┐  │
│  │     CDN       │    │             API Gateway                     │  │
│  │  (CloudFront/ │    │  ┌────────────┐ ┌─────────┐ ┌───────────┐  │  │
│  │   Cloudflare) │    │  │Rate Limiting│ │  Auth   │ │  Routing  │  │  │
│  │               │    │  └────────────┘ │(Token   │ └───────────┘  │  │
│  │  Static Assets│    │  ┌────────────┐ │Validate)│ ┌───────────┐  │  │
│  │  JS/CSS/Img   │    │  │    CORS    │ └─────────┘ │SSL Termin.│  │  │
│  └──────────────┘    │  └────────────┘              └───────────┘  │  │
│                       └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         BFF LAYER (DMZ)                                │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                      Portal BFF                                  │  │
│  │  ┌──────────────────┐  ┌──────────────┐  ┌───────────────────┐   │  │
│  │  │  API Aggregation │  │ Response     │  │ WebSocket Proxy   │   │  │
│  │  │  (Parallel calls)│  │ Transform   │  │ (Notifications)   │   │  │
│  │  └──────────────────┘  └──────────────┘  └───────────────────┘   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      SERVICE LAYER (Private)                           │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────────┐   │
│  │   Auth     │ │  MDM Hub   │ │App Registry│ │  Notification    │   │
│  │  Service   │ │  Service   │ │  Service   │ │    Service       │   │
│  │            │ │            │ │            │ │                  │   │
│  │ - Login    │ │ - CRUD     │ │ - Register │ │ - Push/Email     │   │
│  │ - Token    │ │ - Approval │ │ - Discovery│ │ - WebSocket      │   │
│  │ - RBAC     │ │ - Version  │ │ - Health   │ │ - Template       │   │
│  │ - Session  │ │ - Lookup   │ │ - Manifest │ │                  │   │
│  └────────────┘ └────────────┘ └────────────┘ └──────────────────┘   │
│  ┌────────────┐                                                       │
│  │   Audit    │                                                       │
│  │  Service   │                                                       │
│  │            │                                                       │
│  │ - Log      │                                                       │
│  │ - Query    │                                                       │
│  │ - Report   │                                                       │
│  └────────────┘                                                       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                       DATA LAYER (Private)                             │
│  ┌──────────────────────────────────────────────────────────┐          │
│  │                    PostgreSQL Cluster                    │          │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐           │          │
│  │  │ Auth DB  │    │  MDM DB  │    │ Audit DB │           │          │
│  │  └──────────┘    └──────────┘    └──────────┘           │          │
│  └──────────────────────────────────────────────────────────┘          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐           │
│  │    Redis     │  │    Kafka     │  │  Elasticsearch    │           │
│  │   (Cache)    │  │   (Events)   │  │    (Search)       │           │
│  │              │  │              │  │                   │           │
│  │ - Session    │  │ - MDM Change │  │ - Audit Log      │           │
│  │ - Code Cache │  │ - Audit Event│  │ - Code Search    │           │
│  │ - Rate Limit │  │ - Notification│ │ - Full-text      │           │
│  └──────────────┘  └──────────────┘  └───────────────────┘           │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                     CROSS-CUTTING CONCERNS                             │
│  ┌─────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │      Observability          │  │      Secret Management          │  │
│  │  ┌───────────────────────┐  │  │  ┌───────────────────────────┐  │  │
│  │  │   OpenTelemetry       │  │  │  │   HashiCorp Vault         │  │  │
│  │  │   Collector           │  │  │  │                           │  │  │
│  │  │   ┌─────┐ ┌────────┐ │  │  │  │   - DB Credentials        │  │  │
│  │  │   │Logs │ │Metrics │ │  │  │  │   - API Keys              │  │  │
│  │  │   │(ELK)│ │(Prom.) │ │  │  │  │   - JWT Signing Keys      │  │  │
│  │  │   └─────┘ └────────┘ │  │  │  │   - Encryption Keys       │  │  │
│  │  │   ┌──────────┐       │  │  │  │                           │  │  │
│  │  │   │ Traces   │       │  │  │  │   Auto-rotation Enabled   │  │  │
│  │  │   │ (Jaeger) │       │  │  │  └───────────────────────────┘  │  │
│  │  │   └──────────┘       │  │  └─────────────────────────────────┘  │
│  │  └───────────────────────┘  │                                      │
│  └─────────────────────────────┘                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

**Fowler:** 이 다이어그램에서 핵심 설계 원칙 세 가지를 강조하고 싶습니다.

**Kleppmann:** 통신 패턴부터 정리해 봅시다. 세 가지 유형이 필요합니다.

**Fowler:** 맞습니다. 정리하면 이렇습니다.

### 통신 패턴

| 유형 | 패턴 | 용도 | 경로 |
|------|------|------|------|
| 동기 (Sync) | REST API | Gateway에서 서비스 호출 | Gateway -> BFF -> Services |
| 비동기 (Async) | Kafka Events | MDM 변경, 감사 이벤트, 알림 | Services -> Kafka -> Consumers |
| 실시간 (Real-time) | WebSocket | 포털 알림 전달 | BFF -> Browser (via WS) |

**Kleppmann:** Kafka 토픽 설계에 대해 한 가지 덧붙이면, 이벤트 토픽을 도메인별로 분리해야 합니다. `mdm.code.changed`, `auth.permission.updated`, `audit.event.created`, `notification.dispatch` 이런 식으로요. 토픽 파티셔닝은 tenant ID 또는 project ID 기준으로 합니다.

**Schneier:** 네트워크 세그먼트 분리도 중요합니다. 보안 관점에서 세 영역으로 나눕시다.

### 네트워크 세그먼트

| 세그먼트 | 구성 요소 | 접근 제어 |
|----------|-----------|-----------|
| Public | CDN, API Gateway | 인터넷 노출, WAF 보호 |
| DMZ | BFF | Gateway에서만 접근 가능 |
| Private | Services, DBs, Kafka, Redis, ES | BFF 및 내부 서비스에서만 접근 |

**Schneier:** Private 세그먼트의 DB는 절대 외부에서 직접 접근할 수 없어야 합니다. 서비스 간 통신도 mTLS로 암호화합니다. API Gateway가 유일한 외부 진입점이 되어야 합니다.

**Ng:** 향후 AI/ML 서비스가 추가될 경우, Private 세그먼트 내에 별도의 ML Service Layer를 둘 수 있겠습니다. MDM의 데이터 품질 분석이나 이상 탐지 같은 기능이요.

**Beck:** 좋습니다. 전체 그림이 명확해졌습니다. 다음으로 API Gateway와 BFF 설계를 구체화합시다.

---

## 토의 5.2: API Gateway / BFF 설계

**Beck:** Gateway와 BFF의 역할 분담을 명확히 해야 합니다. Fowler와 Uncle Bob이 이끌어 주시죠.

**Fowler:** API Gateway와 BFF는 서로 다른 관심사를 처리합니다. Gateway는 인프라 수준의 공통 관심사, BFF는 클라이언트 최적화를 담당합니다.

### API Gateway 책임

**Uncle Bob:** Gateway의 책임을 Clean Architecture 관점에서 정리하면, Gateway는 외부 세계와 내부 시스템의 경계입니다. 구체적인 책임은 다음과 같습니다.

| 책임 | 설명 | 구현 세부사항 |
|------|------|---------------|
| 라우팅 | 요청을 적절한 BFF 또는 서비스로 전달 | Path-based routing, header-based routing |
| Rate Limiting | 과도한 요청 차단 | Token bucket 알고리즘, Redis 기반 분산 카운터 |
| 인증 | JWT 토큰 검증 (발급은 Auth Service) | JWT signature verification, expiry check |
| 요청 로깅 | 모든 요청/응답 메타데이터 기록 | Trace ID 생성, 응답 시간 기록 |
| CORS | Cross-Origin 정책 관리 | 허용 도메인 화이트리스트 |
| SSL 종료 | HTTPS 처리 | TLS 1.3, 인증서 자동 갱신 |

**Fowler:** Gateway 기술 옵션을 비교해 봅시다.

### Gateway 기술 옵션 비교

| 옵션 | 장점 | 단점 | 적합도 |
|------|------|------|--------|
| Kong | 플러그인 생태계 풍부, 검증됨 | Lua 커스터마이징 러닝 커브 | 높음 |
| Apache APISIX | 고성능, 동적 라우팅 | 상대적으로 작은 커뮤니티 | 중간 |
| Custom (Node.js) | 완전한 제어, 팀 친화적 | 유지보수 부담, 보안 취약점 | 낮음 |
| Custom (Go) | 고성능, 완전한 제어 | 개발 기간 길어짐 | 중간 |

**Uncle Bob:** 저는 Kong을 추천합니다. 플러그인으로 대부분의 요구사항을 충족하고, 커스텀 플러그인도 작성할 수 있습니다. 처음부터 Gateway를 직접 만드는 것은 Not Invented Here 증후군입니다.

**Schneier:** 동의합니다. Gateway는 보안의 최전선이므로 검증된 솔루션을 사용해야 합니다. 직접 만들면 보안 취약점이 생기기 쉽습니다.

### Rate Limiting 규칙

**Schneier:** Rate limiting 규칙은 엔드포인트별로 차등 적용해야 합니다. 특히 인증 관련 엔드포인트는 브루트 포스 공격에 노출되므로 더 엄격해야 합니다.

```yaml
# rate-limiting-config.yaml
rate_limits:
  # 인증 엔드포인트: 브루트 포스 방지
  auth_endpoints:
    path_pattern: "/api/auth/**"
    limit: 10
    window: "1m"
    key: "client_ip"
    retry_after_header: true
    block_duration: "5m"  # 초과 시 5분 차단

  # 일반 API: 사용자별 제한
  general_apis:
    path_pattern: "/api/**"
    limit: 100
    window: "1m"
    key: "user_id"
    burst_allowance: 20  # 순간 버스트 허용

  # Lookup API: 높은 빈도 예상 (코드 조회 등)
  lookup_apis:
    path_pattern: "/api/master/lookup/**"
    limit: 1000
    window: "1m"
    key: "user_id"
    cache_hint: true  # 클라이언트 캐시 권장 헤더

  # 관리자 API: 더 높은 제한
  admin_apis:
    path_pattern: "/api/admin/**"
    limit: 200
    window: "1m"
    key: "user_id"
    required_role: "SYSTEM_ADMIN"
```

**Uncle Bob:** IP 기반 제한만으로는 부족합니다. 프록시 뒤에 여러 사용자가 있을 수 있어요. 인증 전은 IP 기반, 인증 후는 user_id 기반으로 이중 적용합니다.

### BFF 패턴

**Fowler:** BFF(Backend for Frontend) 패턴의 핵심은 프론트엔드가 필요로 하는 형태로 데이터를 조합하는 것입니다. 프론트엔드가 여러 마이크로서비스를 직접 호출하면 chatty API 문제가 생기죠.

**Uncle Bob:** 구체적인 예시로 설명하겠습니다. 대시보드 페이지를 로드할 때, 프론트엔드가 필요한 데이터는 사용자 정보, 최근 프로젝트, 알림, 승인 대기 항목, 마스터 데이터 통계입니다. BFF 없이는 5개의 API를 순차 또는 병렬 호출해야 합니다.

```typescript
// BFF endpoint example
// GET /bff/dashboard
interface DashboardResponse {
  user: UserInfo;
  recentProjects: ProjectSummary[];
  notifications: Notification[];
  pendingApprovals: ApprovalSummary[];
  masterDataStats: { totalCodes: number; recentChanges: number };
}

// BFF 내부 구현 (간략)
async function getDashboard(userId: string): Promise<DashboardResponse> {
  // 모든 서비스 호출을 병렬로 실행
  const [user, projects, notifications, approvals, stats] = await Promise.all([
    authService.getUser(userId),
    registryService.getRecentProjects(userId),
    notificationService.getUnread(userId, { limit: 10 }),
    authService.getPendingApprovals(userId),
    mdmService.getStats(),
  ]);

  return {
    user: transformUserInfo(user),
    recentProjects: projects.map(toProjectSummary),
    notifications: notifications.items,
    pendingApprovals: approvals.map(toApprovalSummary),
    masterDataStats: {
      totalCodes: stats.totalActiveCodeCount,
      recentChanges: stats.changesLast24h,
    },
  };
}
```

**Fowler:** GraphQL에 대해서도 논의해 봤는데요, REST + BFF 조합이 시작하기에는 더 단순합니다.

**Karpathy:** GraphQL은 클라이언트가 매우 다양한 데이터 조합을 필요로 할 때 빛을 발합니다. 현재는 포털 하나의 BFF로 충분하고, 모바일 등 다른 클라이언트가 추가되면 그때 도입을 고려하는 게 맞습니다.

**Nielsen:** UX 관점에서 BFF가 응답을 조합할 때, 부분 실패를 gracefully 처리해야 합니다. 예를 들어 알림 서비스가 응답하지 않더라도 대시보드의 나머지 영역은 정상 표시되어야 합니다.

**Uncle Bob:** 맞습니다. 각 서비스 호출에 개별 timeout과 fallback을 설정합니다. Circuit breaker 패턴을 적용해서 하나의 서비스 장애가 전체 BFF를 다운시키지 않도록 합니다.

```typescript
// 부분 실패 처리 BFF 패턴
async function getDashboardResilient(userId: string): Promise<DashboardResponse> {
  const results = await Promise.allSettled([
    withTimeout(authService.getUser(userId), 2000),
    withTimeout(registryService.getRecentProjects(userId), 3000),
    withTimeout(notificationService.getUnread(userId), 2000),
    withTimeout(authService.getPendingApprovals(userId), 3000),
    withTimeout(mdmService.getStats(), 2000),
  ]);

  return {
    user: getOrDefault(results[0], null),
    recentProjects: getOrDefault(results[1], []),
    notifications: getOrDefault(results[2], []),
    pendingApprovals: getOrDefault(results[3], []),
    masterDataStats: getOrDefault(results[4], { totalCodes: 0, recentChanges: 0 }),
  };
}
```

**Beck:** BFF에서 부분 실패를 허용하되, 어떤 데이터가 실패했는지 프론트엔드에 알려주는 메타 정보도 포함시키면 좋겠습니다. UX에서 해당 영역에 "다시 시도" 버튼을 표시할 수 있으니까요.

---

## 토의 5.3: 공통 SDK 명세

**Beck:** SDK는 외부 팀이 우리 플랫폼과 통합하는 주요 접점입니다. Uncle Bob이 설계를 이끌어 주세요.

**Uncle Bob:** SDK 설계의 핵심 원칙은 세 가지입니다. 첫째, 단순한 인터페이스. 둘째, 내부 복잡성 캡슐화. 셋째, 실패에 대한 명확한 피드백. 개발자가 인증 토큰 갱신이나 캐시 무효화 같은 인프라 로직을 직접 다루면 안 됩니다.

### Auth SDK

```typescript
// @portal/auth-sdk
// 지원 언어: TypeScript, Java, Python, Go

interface AuthSDK {
  // ── 초기화 ──
  init(config: AuthConfig): Promise<void>;

  // ── 인증 (Authentication) ──
  login(returnUrl?: string): void;          // SSO 페이지로 리다이렉트
  logout(): Promise<void>;                  // 세션 종료 + 토큰 폐기
  getUser(): UserInfo | null;               // 현재 로그인 사용자
  getToken(): string | null;                // 현재 유효한 Access Token
  isAuthenticated(): boolean;               // 인증 여부 확인
  onAuthStateChange(callback: (user: UserInfo | null) => void): Unsubscribe;

  // ── 인가 (Authorization) ──
  hasPermission(permission: string, projectId?: string): boolean;
  hasRole(role: string, projectId?: string): boolean;
  hasAnyPermission(permissions: string[], projectId?: string): boolean;

  // ── 토큰 관리 (내부, 자동 처리) ──
  refreshToken(): Promise<void>;
}

interface AuthConfig {
  authServerUrl: string;       // 인증 서버 URL
  clientId: string;            // 등록된 클라이언트 ID
  redirectUri: string;         // 로그인 후 콜백 URL
  scopes: string[];            // 요청할 권한 스코프
}

interface UserInfo {
  userId: string;
  email: string;
  name: string;
  roles: RoleAssignment[];
  permissions: string[];
  lastLogin: string;
  profileImageUrl?: string;
}

interface RoleAssignment {
  roleCode: string;
  roleName: string;
  projectId?: string;          // null이면 글로벌 역할
  expiresAt?: string;
}

type Unsubscribe = () => void;
```

**Uncle Bob:** Auth SDK의 핵심 설계 결정을 설명하겠습니다.

첫째, `login()` 메서드는 void를 반환합니다. SSO 리다이렉트이기 때문에 Promise가 아닙니다. 리다이렉트 후 콜백 URL로 돌아오면 SDK가 자동으로 토큰을 교환합니다.

둘째, `hasPermission()`과 `hasRole()`은 동기(boolean)입니다. 초기화 시 권한 정보를 메모리에 캐시하기 때문입니다. 서버 사이드에서도 동일합니다. 토큰 페이로드에 권한이 포함되어 있으니까요.

셋째, 토큰 갱신은 내부적으로 자동 처리됩니다. Access Token 만료 5분 전에 Refresh Token으로 갱신합니다. 개발자는 이 과정을 인지할 필요가 없습니다.

**Schneier:** `getToken()`을 외부에 노출하는 것은 의도적인 결정인가요? 토큰이 잘못 사용될 수 있습니다.

**Uncle Bob:** 맞는 지적입니다. 하지만 SDK 외부의 HTTP 클라이언트에서 토큰이 필요한 경우가 있습니다. 대안으로 `createAuthenticatedFetch()` 같은 래퍼를 제공해서 토큰을 직접 다루지 않아도 되게 합니다. `getToken()`은 탈출구로 남겨두되 문서에서 주의사항을 명시합니다.

```typescript
// 권장: SDK가 제공하는 인증된 HTTP 클라이언트 사용
const authFetch = authSDK.createAuthenticatedFetch();
const response = await authFetch('/api/projects', { method: 'GET' });
// 토큰 자동 첨부, 만료 시 자동 갱신, 401 시 자동 재시도
```

### Master SDK

```typescript
// @portal/master-sdk
// 지원 언어: TypeScript, Java, Python, Go

interface MasterSDK {
  init(config: MasterConfig): Promise<void>;

  // ── 코드 조회 (캐시 적용) ──
  getCode(groupCode: string, codeValue: string, locale?: string): CodeInfo | null;
  getCodeList(groupCode: string, locale?: string): CodeInfo[];
  getCodeName(groupCode: string, codeValue: string, locale?: string): string;

  // ── 유효성 검증 ──
  isValidCode(groupCode: string, codeValue: string): boolean;

  // ── 변경 구독 ──
  onCodeChange(groupCode: string, callback: (event: CodeChangeEvent) => void): Unsubscribe;

  // ── 캐시 제어 ──
  invalidateCache(groupCode?: string): void;   // 특정 그룹 또는 전체 캐시 무효화
  preload(groupCodes: string[]): Promise<void>; // 지정한 그룹 코드를 미리 로드
}

interface MasterConfig {
  apiUrl: string;               // MDM Hub API URL
  cacheStrategy: 'memory' | 'localStorage' | 'sessionStorage';
  cacheTTL: number;             // 캐시 유효 시간 (초), 기본값 300
  defaultLocale: string;        // 기본 로케일 (예: 'ko')
  enableRealTimeSync: boolean;  // Kafka/WebSocket 기반 실시간 동기화
}

interface CodeInfo {
  groupCode: string;
  codeValue: string;
  codeName: string;
  codeNameEn?: string;
  description?: string;
  sortOrder: number;
  isActive: boolean;
  parentCodeValue?: string;
  attributes: Record<string, string>;
}

interface CodeChangeEvent {
  type: 'CREATED' | 'UPDATED' | 'DELETED' | 'ACTIVATED' | 'DEACTIVATED';
  groupCode: string;
  codeValue: string;
  previousVersion?: CodeInfo;
  currentVersion?: CodeInfo;
  changedBy: string;
  changedAt: string;
}
```

**Kleppmann:** Master SDK의 캐시 전략에 대해 말씀드리겠습니다. 코드 데이터는 변경 빈도가 낮으므로 적극적으로 캐시해야 합니다. 하지만 캐시 일관성 문제가 있습니다. 두 가지 전략을 조합합니다.

첫째, TTL 기반 캐시. 기본 5분입니다. 대부분의 경우 이것으로 충분합니다.

둘째, 이벤트 기반 캐시 무효화. `enableRealTimeSync: true`로 설정하면 Kafka 이벤트를 WebSocket으로 전달받아 즉시 캐시를 갱신합니다. 이것은 승인 워크플로우 후 변경이 즉시 반영되어야 하는 경우에 사용합니다.

**Uncle Bob:** `preload()`는 앱 초기화 시 자주 사용하는 코드 그룹을 미리 로드하는 용도입니다. 화면 진입 시 코드 조회 지연을 없앱니다.

```typescript
// 앱 초기화 시 preload 사용 예시
await masterSDK.init(config);
await masterSDK.preload([
  'PROJECT_STATUS',
  'USER_ROLE_TYPE',
  'APPROVAL_STATUS',
  'COUNTRY_CODE',
  'CURRENCY_CODE'
]);

// 이후 getCode / getCodeList는 캐시에서 즉시 반환
const statusName = masterSDK.getCodeName('PROJECT_STATUS', 'ACTIVE', 'ko');
// => "활성"
```

### Portal SDK (프론트엔드 전용)

```typescript
// @portal/portal-sdk
// 프론트엔드(브라우저)에서만 사용

interface PortalSDK {
  // ── 앱 등록 ──
  registerApp(manifest: AppManifest): void;

  // ── Shell 통신 ──
  navigate(path: string): void;
  getCurrentPath(): string;

  // ── 이벤트 ──
  emit(event: string, payload: any): void;
  on(event: string, handler: (payload: any) => void): Unsubscribe;

  // ── UI 통합 ──
  showNotification(notification: NotificationConfig): void;
  showModal(config: ModalConfig): Promise<ModalResult>;
  setBreadcrumb(items: BreadcrumbItem[]): void;
  setPageTitle(title: string): void;
}

interface AppManifest {
  name: string;                 // 앱 고유 이름
  displayName: string;          // 표시 이름
  version: string;              // 앱 버전
  basePath: string;             // 라우팅 기본 경로 (예: '/mdm')
  requiredPermissions: string[];// 앱 접근에 필요한 권한
  mountPoint: string;           // DOM 마운트 포인트 ID
  lifecycle: {
    mount: (container: HTMLElement, props: MountProps) => void;
    unmount: (container: HTMLElement) => void;
    update?: (props: MountProps) => void;
  };
}

interface MountProps {
  authSDK: AuthSDK;
  masterSDK: MasterSDK;
  portalSDK: PortalSDK;
  basePath: string;
  locale: string;
}

interface NotificationConfig {
  type: 'success' | 'error' | 'warning' | 'info';
  title: string;
  message: string;
  duration?: number;             // 자동 닫힘 시간 (ms), 기본 5000
  action?: { label: string; onClick: () => void };
}

interface ModalConfig {
  title: string;
  content: string | HTMLElement;
  size?: 'small' | 'medium' | 'large';
  confirmText?: string;
  cancelText?: string;
  dangerous?: boolean;           // true이면 확인 버튼이 빨간색
}

interface ModalResult {
  confirmed: boolean;
  data?: any;
}

interface BreadcrumbItem {
  label: string;
  path?: string;                 // 클릭 가능한 경우
}
```

**Nielsen:** Portal SDK의 UI 통합 인터페이스에 대해 두 가지 제안이 있습니다. 첫째, `showNotification()`의 duration이 0이면 사용자가 직접 닫기 전까지 유지되어야 합니다. 에러 메시지는 자동으로 사라지면 안 됩니다. 둘째, `showModal()`에 키보드 포커스 트랩이 내장되어야 합니다. 접근성(a11y) 필수 요구사항입니다.

**Uncle Bob:** 맞습니다. SDK 내부에서 처리하므로 개발자가 신경 쓸 필요가 없습니다.

### SDK 배포 및 버전 관리

| 언어 | 배포 채널 | 패키지명 |
|------|----------|---------|
| TypeScript | npm | `@portal/auth-sdk`, `@portal/master-sdk`, `@portal/portal-sdk` |
| Java | Maven Central | `com.portal:auth-sdk`, `com.portal:master-sdk` |
| Python | PyPI | `portal-auth-sdk`, `portal-master-sdk` |
| Go | Go Modules | `github.com/portal/auth-sdk-go`, `github.com/portal/master-sdk-go` |

**Uncle Bob:** 버전 관리 정책은 Semantic Versioning을 따릅니다. 하위 호환성은 2개 메이저 버전까지 보장합니다. 예를 들어, v3.x가 출시되면 v1.x는 EOL이 되고 v2.x는 보안 패치만 제공합니다.

```
// 버전 호환성 매트릭스
v1.x - 보안 패치만 (v3 출시 시 EOL)
v2.x - 활성 유지보수 (현재 안정 버전)
v3.x - 최신 개발 버전

// 주요 변경 시 deprecation 워닝 선행
@deprecated("Use hasPermission() instead. Will be removed in v4.0")
checkPermission(perm: string): boolean;
```

**Beck:** SDK가 잘 정리됐습니다. 이 SDK들이 잘 작동하려면 CI/CD 파이프라인이 견고해야 합니다.

---

## 토의 5.4: CI/CD 파이프라인 설계

**Beck:** CI/CD는 개발 속도와 품질의 균형점입니다. Uncle Bob과 함께 설계해 봅시다.

**Uncle Bob:** CI/CD 파이프라인은 컴포넌트별로 독립적이어야 합니다. 모놀리식 파이프라인은 하나의 변경이 전체 배포를 블로킹합니다.

### 컴포넌트별 파이프라인

**Portal Shell 파이프라인:**

```
┌──────┐   ┌───────────┐   ┌───────┐   ┌──────────────────┐
│ Lint │──>│ Unit Test │──>│ Build │──>│ Integration Test │
└──────┘   └───────────┘   └───────┘   └──────────────────┘
                                                │
                                                ▼
                                   ┌─────────────────────┐
                                   │  Deploy to Staging   │
                                   └─────────────────────┘
                                                │
                                                ▼
                                   ┌─────────────────────┐
                                   │    Smoke Test        │
                                   └─────────────────────┘
                                                │
                                                ▼
                                   ┌─────────────────────┐
                                   │  Deploy to Prod      │
                                   │  (Canary 10% -> 100%)│
                                   └─────────────────────┘
```

**Remote App 파이프라인:**

```
┌──────┐   ┌───────────┐   ┌───────┐   ┌───────────────┐
│ Lint │──>│ Unit Test │──>│ Build │──>│ Contract Test │
└──────┘   └───────────┘   └───────┘   └───────────────┘
                                                │
                                                ▼
                                   ┌──────────────────────┐
                                   │  Integration Test     │
                                   └──────────────────────┘
                                                │
                                                ▼
                                   ┌──────────────────────┐
                                   │ Portal Integration   │
                                   │ Test (Shell 내 마운트) │ <── ★ 필수 게이트
                                   └──────────────────────┘
                                                │
                                                ▼
                                   ┌──────────────────────┐
                                   │    Deploy to Prod     │
                                   └──────────────────────┘
```

**SDK 파이프라인:**

```
┌──────┐   ┌───────────┐   ┌───────┐   ┌───────────────────┐
│ Lint │──>│ Unit Test │──>│ Build │──>│ Backward Compat   │
└──────┘   └───────────┘   └───────┘   │      Test         │
                                        └───────────────────┘
                                                │
                                                ▼
                                   ┌──────────────────────┐
                                   │  Publish to Registry  │
                                   │  (npm/Maven/PyPI/Go)  │
                                   └──────────────────────┘
```

**서비스 파이프라인 (Auth, MDM Hub, etc.):**

```
┌──────┐   ┌───────────┐   ┌───────┐   ┌───────────────┐
│ Lint │──>│ Unit Test │──>│ Build │──>│ Contract Test │
└──────┘   └───────────┘   └───────┘   └───────────────┘
                                                │
                                                ▼
                                   ┌──────────────────────┐
                                   │  Integration Test     │
                                   │  (Service+DB+Cache)   │
                                   └──────────────────────┘
                                                │
                                                ▼
                                   ┌──────────────────────┐
                                   │   Security Scan       │
                                   │ (SAST+DAST+Dep+Image) │
                                   └──────────────────────┘
                                                │
                                                ▼
                                   ┌──────────────────────┐
                                   │  Deploy (Blue-Green)  │
                                   └──────────────────────┘
```

### 통합 테스트 게이트

**Uncle Bob:** Remote App은 반드시 Portal Integration Test를 통과해야 프로덕션 배포가 가능합니다. 이 테스트는 실제 Portal Shell 환경에서 Remote App이 올바르게 마운트되고, SDK 통신이 정상 작동하는지 검증합니다.

**Beck:** 이 게이트가 병목이 되지 않도록 주의해야 합니다. 테스트를 경량화하고 병렬 실행이 가능하도록 설계합시다.

### Contract Testing

**Uncle Bob:** Consumer-Driven Contract Testing에 Pact를 사용합니다.

```
┌──────────────┐         ┌─────────────┐         ┌──────────────┐
│  Consumer    │         │    Pact     │         │  Provider    │
│ (BFF/App)    │──Push──>│   Broker    │<──Verify│  (Service)   │
│              │         │             │         │              │
│  계약 생성    │         │  계약 저장   │         │  계약 검증    │
└──────────────┘         └─────────────┘         └──────────────┘
```

- Consumer(BFF 또는 Remote App)가 기대하는 API 계약을 정의합니다.
- Provider(Service)가 자신의 CI에서 해당 계약을 검증합니다.
- 계약이 깨지면 Provider 배포가 차단됩니다.

### 보안 스캐닝

| 스캔 유형 | 도구 | 적용 시점 | 차단 기준 |
|----------|------|----------|----------|
| SAST (정적 분석) | SonarQube | PR 머지 전 | Critical/High 이슈 0건 |
| DAST (동적 분석) | OWASP ZAP | Staging 배포 후 | High 이슈 0건 |
| 의존성 스캔 | Snyk | 빌드 시 + 주간 정기 | Known Critical CVE 0건 |
| 컨테이너 스캔 | Trivy | 이미지 빌드 후 | Critical CVE 0건 |

### 배포 전략

**Fowler:** 배포 전략은 컴포넌트 특성에 따라 달라야 합니다.

| 컴포넌트 | 전략 | 이유 |
|---------|------|------|
| Backend Services | Blue-Green | 즉각적인 롤백 필요, DB 마이그레이션 호환성 |
| Portal Frontend | Canary (10% -> 25% -> 50% -> 100%) | 사용자 영향도 점진적 확인 |
| SDK | 레지스트리 발행 | 소비자가 업그레이드 시점 선택 |

**Uncle Bob:** 롤백 시간은 최대 5분 이내여야 합니다. Blue-Green에서는 이전 환경이 즉시 활성화됩니다. Canary에서는 라우팅 비율을 0%로 되돌립니다.

### 환경 관리

```
┌─────┐    ┌──────────┐    ┌─────────────┐
│ Dev │───>│ Staging  │───>│ Production  │
└─────┘    └──────────┘    └─────────────┘
                                 │
                          Feature Flags
                          (LaunchDarkly/
                           Unleash)
```

- **Dev:** 개발자 자유롭게 배포, 자동화된 테스트 실행
- **Staging:** Production 미러, 통합 테스트 및 UAT
- **Production:** Feature flag로 점진적 롤아웃

**Beck:** Feature flag를 통해 코드는 배포하되 기능은 숨길 수 있습니다. 새로운 MDM 승인 워크플로우 같은 큰 기능은 flag 뒤에 두고 검증 후 활성화합니다.

---

## 토의 5.5: 관찰가능성 설계 (Logging, Metrics, Tracing)

**Beck:** 시스템이 운영 중일 때 무슨 일이 일어나고 있는지 알 수 없다면, 아무리 좋은 아키텍처도 소용없습니다. Fowler, 관찰가능성 설계를 이끌어 주세요.

**Fowler:** 관찰가능성의 세 기둥인 Logging, Metrics, Tracing 각각을 설계하겠습니다.

### Logging

```typescript
// 구조화된 로그 표준 포맷
interface StructuredLog {
  timestamp: string;     // ISO 8601 (예: "2025-03-15T09:30:00.123Z")
  level: 'debug' | 'info' | 'warn' | 'error';
  service: string;       // 서비스 이름 (예: "auth-service")
  traceId: string;       // 분산 추적 ID
  spanId: string;        // 현재 span ID
  userId?: string;       // 요청한 사용자 (있는 경우)
  projectId?: string;    // 관련 프로젝트 (있는 경우)
  message: string;       // 사람이 읽을 수 있는 메시지
  context: Record<string, any>;  // 추가 컨텍스트
}
```

**Uncle Bob:** 로그 포맷은 모든 서비스에서 통일되어야 합니다. SDK에서 로거 팩토리를 제공해서 이 포맷을 강제합니다.

```typescript
// 로거 사용 예시
import { createLogger } from '@portal/observability';

const logger = createLogger({ service: 'mdm-hub-service' });

logger.info('Master code created', {
  traceId: ctx.traceId,
  userId: ctx.userId,
  context: {
    groupCode: 'PROJECT_STATUS',
    codeValue: 'ACTIVE',
    action: 'CREATE',
  },
});

// 출력:
// {"timestamp":"2025-03-15T09:30:00.123Z","level":"info",
//  "service":"mdm-hub-service","traceId":"abc123","spanId":"def456",
//  "userId":"user-001","message":"Master code created",
//  "context":{"groupCode":"PROJECT_STATUS","codeValue":"ACTIVE","action":"CREATE"}}
```

**Schneier:** 로그에 PII(개인식별정보)를 절대 포함하면 안 됩니다. 이메일, 전화번호, 비밀번호 같은 민감 필드는 마스킹해야 합니다. SDK 로거에 자동 마스킹 기능을 내장하세요.

```typescript
// 자동 PII 마스킹
const SENSITIVE_FIELDS = ['password', 'email', 'phone', 'ssn', 'token'];

function maskSensitiveData(data: Record<string, any>): Record<string, any> {
  return Object.entries(data).reduce((acc, [key, value]) => {
    if (SENSITIVE_FIELDS.some(f => key.toLowerCase().includes(f))) {
      acc[key] = '***MASKED***';
    } else if (typeof value === 'object' && value !== null) {
      acc[key] = maskSensitiveData(value);
    } else {
      acc[key] = value;
    }
    return acc;
  }, {} as Record<string, any>);
}
```

### 로그 수집 및 보관

| 구성 요소 | 기술 | 비고 |
|----------|------|------|
| 수집 | Fluentd / Fluent Bit (사이드카) | 컨테이너 로그 수집 |
| 저장 및 검색 | ELK Stack 또는 Loki + Grafana | 풀텍스트 검색 |
| 시각화 | Kibana / Grafana | 대시보드 및 검색 UI |

**로그 보관 정책:**

| 보관 등급 | 기간 | 저장소 | 용도 |
|----------|------|--------|------|
| Hot | 30일 | Elasticsearch (SSD) | 실시간 검색 및 분석 |
| Warm | 90일 | Elasticsearch (HDD) | 과거 이슈 조사 |
| Cold | 1년 | S3 / Object Storage | 컴플라이언스, 감사 |

### Metrics

**Fowler:** 모든 서비스에 Golden Signals(4가지 황금 신호)를 적용합니다.

| 신호 | 메트릭 | 임계값 (SLO) |
|------|--------|-------------|
| Latency | p50, p95, p99 응답 시간 | p95 < 200ms, p99 < 500ms |
| Traffic | 초당 요청 수 (RPS) | 모니터링 (이상 탐지) |
| Errors | 5xx 에러 비율 | < 0.1% |
| Saturation | CPU, Memory, DB Connection Pool | CPU < 70%, Memory < 80% |

**비즈니스 메트릭:**

| 메트릭 | 설명 | 대시보드 |
|--------|------|---------|
| active_users_count | 현재 활성 사용자 수 | 운영 대시보드 |
| login_count_per_minute | 분당 로그인 횟수 | 보안 대시보드 |
| permission_checks_per_sec | 초당 권한 검증 횟수 | Auth 서비스 대시보드 |
| master_code_lookups_per_sec | 초당 마스터 코드 조회 | MDM 서비스 대시보드 |
| cache_hit_ratio | 캐시 적중률 | 성능 대시보드 |
| approval_pending_count | 승인 대기 건수 | 비즈니스 대시보드 |
| api_gateway_rejection_rate | Gateway 거부율 | 보안 대시보드 |

**Karpathy:** 비즈니스 메트릭에 이상 탐지를 적용하면 좋겠습니다. 갑자기 로그인 실패율이 급증하면 보안 공격일 수 있고, 코드 조회가 급증하면 캐시 무효화 이슈일 수 있습니다. 이런 이상 패턴을 자동으로 탐지하는 ML 모델을 나중에 붙일 수 있습니다.

**Fowler:** SLO 대시보드는 서비스별로 구성합니다. Error Budget 소진율을 시각화해서 팀이 안정성과 속도 사이의 균형을 판단하도록 합니다.

### 알림 규칙 (Alerting)

```yaml
# alerting-rules.yaml
groups:
  - name: service-slo
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.001
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.service }} 에러율 0.1% 초과"

      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 0.2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.service }} p95 레이턴시 200ms 초과"

      - alert: AuthServiceDown
        expr: up{service="auth-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Auth Service 다운"

# 에스컬레이션 정책
escalation:
  - severity: warning
    channels: [slack_dev_channel]
    wait: 15m
  - severity: critical
    channels: [slack_ops_channel, pagerduty]
    wait: 5m
  - severity: critical
    if_not_acked: 15m
    channels: [phone_call_oncall_lead]
```

### Tracing (분산 추적)

**Fowler:** 분산 추적은 OpenTelemetry 표준을 사용합니다.

```
Frontend (Browser)
    │  traceparent: 00-traceId-spanId-01
    ▼
API Gateway
    │  W3C Trace Context 헤더 전파
    ▼
Portal BFF ──────────────────────────────────────┐
    │                                             │
    ├──> Auth Service ──> Auth DB                 │
    │         (span: auth.getUser)                │
    │                                             │
    ├──> MDM Hub Service ──> MDM DB ──> Redis     │
    │         (span: mdm.getStats)                │
    │                                             │
    └──> Notification Service                     │
              (span: notification.getUnread)      │
                                                  │
    전체 trace가 하나의 traceId로 연결됨            │
```

**추적 컨텍스트 포맷:** W3C Trace Context (`traceparent` 헤더)

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
             │   │                                │                 │
             │   traceId (32 hex)                  spanId (16 hex)   flags
             version
```

**샘플링 전략:**

| 상황 | 샘플링 비율 | 이유 |
|------|-----------|------|
| 에러 발생 시 | 100% | 모든 에러를 추적 가능하게 |
| 프로덕션 정상 트래픽 | 10% | 비용 최적화 |
| Staging | 100% | 디버깅 용이 |
| 개발 환경 | 100% | 개발자 편의 |

**Fowler:** 프론트엔드에서도 trace를 시작합니다. 사용자가 버튼을 클릭하면 그 시점부터 trace가 시작되어 Gateway, BFF, Service를 거쳐 DB까지 전체 경로를 추적할 수 있습니다. 이를 통해 "사용자가 느리다고 말했을 때 어디가 병목인지" 정확히 파악할 수 있습니다.

**Ng:** OpenTelemetry 데이터를 장기 저장하면, 시스템 성능 패턴을 학습하는 ML 모델에도 활용할 수 있습니다. 특정 시간대에 특정 서비스가 느려지는 패턴 같은 것을 자동으로 탐지하는 거죠.

**Beck:** 관찰가능성은 비용이 될 수도 있으므로 샘플링 비율 조절이 중요합니다. 시작은 보수적으로, 문제가 생기면 비율을 올리는 방식이 좋겠습니다.

---

## 토의 5.6: 테스트 전략

**Beck:** 테스트 전략은 제가 늘 중시하는 영역입니다. Uncle Bob, 체계적으로 정리해 주시죠.

**Uncle Bob:** 테스트 피라미드를 기반으로 각 레벨의 전략을 수립합니다.

```
              ┌──────────┐
              │  E2E     │   적지만 핵심 시나리오
              │ (소수)    │
             ┌┴──────────┴┐
             │Integration │   서비스 + 인프라
             │  (중간)     │
            ┌┴────────────┴┐
            │   Contract    │   서비스 간 계약
            │    (중간)      │
           ┌┴──────────────┴┐
           │    Unit Tests   │   많고 빠르게
           │     (다수)       │
           └────────────────┘
```

### Unit Tests

- **대상:** 각 서비스, SDK, 프론트엔드 컴포넌트
- **커버리지 목표:** 80% 이상 (라인 커버리지)
- **실행 시간 목표:** 전체 유닛 테스트 < 5분
- **도구:** Jest (TypeScript), JUnit (Java), pytest (Python), Go test

```typescript
// Unit Test 예시: Auth SDK의 hasPermission
describe('AuthSDK.hasPermission', () => {
  it('글로벌 권한이 있으면 true를 반환해야 한다', () => {
    const sdk = createAuthSDKWithUser({
      permissions: ['USER_READ', 'USER_WRITE'],
      roles: [{ roleCode: 'ADMIN', projectId: null }],
    });
    expect(sdk.hasPermission('USER_READ')).toBe(true);
  });

  it('프로젝트 범위 권한은 projectId를 확인해야 한다', () => {
    const sdk = createAuthSDKWithUser({
      permissions: [],
      roles: [{ roleCode: 'PROJECT_ADMIN', projectId: 'proj-001' }],
    });
    expect(sdk.hasPermission('CODE_APPROVE', 'proj-001')).toBe(true);
    expect(sdk.hasPermission('CODE_APPROVE', 'proj-002')).toBe(false);
  });

  it('인증되지 않은 사용자는 항상 false를 반환해야 한다', () => {
    const sdk = createAuthSDKWithUser(null);
    expect(sdk.hasPermission('USER_READ')).toBe(false);
  });
});
```

### Contract Tests

- **도구:** Pact
- **대상:** BFF <-> 서비스 간, Remote App <-> BFF 간 API 계약
- **실행 시점:** PR 머지 전 필수

```typescript
// Pact Consumer Test 예시 (BFF가 Auth Service를 호출)
describe('BFF -> Auth Service Contract', () => {
  it('GET /users/:id 응답 계약', async () => {
    await provider.addInteraction({
      state: 'user user-001 exists',
      uponReceiving: 'a request for user info',
      withRequest: {
        method: 'GET',
        path: '/users/user-001',
        headers: { Authorization: 'Bearer valid-token' },
      },
      willRespondWith: {
        status: 200,
        body: Matchers.like({
          userId: 'user-001',
          email: Matchers.email(),
          name: Matchers.string('홍길동'),
          roles: Matchers.eachLike({
            roleCode: Matchers.string('ADMIN'),
            projectId: Matchers.nullable(Matchers.string()),
          }),
        }),
      },
    });

    const response = await bffClient.getUser('user-001');
    expect(response.userId).toBe('user-001');
  });
});
```

### Integration Tests

- **대상:** 서비스 + 실제 DB + 캐시 + 메시지 큐
- **환경:** Docker Compose로 구성된 테스트 인프라
- **실행 시점:** CI 파이프라인, PR 머지 전

```typescript
// Integration Test 예시: MDM Hub Service
describe('MDM Hub Integration', () => {
  beforeAll(async () => {
    // Testcontainers로 PostgreSQL + Redis 시작
    postgresContainer = await new PostgreSQLContainer().start();
    redisContainer = await new GenericContainer('redis:7').start();
    mdmService = createMDMService({ db: postgresContainer, cache: redisContainer });
  });

  it('코드 생성 -> 캐시 저장 -> 조회 시 캐시 히트', async () => {
    // 1. 코드 생성
    const code = await mdmService.createCode({
      groupCode: 'TEST_STATUS',
      codeValue: 'ACTIVE',
      codeName: '활성',
    });

    // 2. 첫 조회: DB에서 가져와서 캐시에 저장
    const first = await mdmService.getCode('TEST_STATUS', 'ACTIVE');
    expect(first.codeName).toBe('활성');

    // 3. 두 번째 조회: 캐시에서 가져옴
    const cacheHit = await redisContainer.get('mdm:TEST_STATUS:ACTIVE');
    expect(cacheHit).not.toBeNull();
  });
});
```

### Portal Integration Tests

- **대상:** Portal Shell + Remote App 마운트 및 통신
- **환경:** 실제 Portal Shell 환경에서 Remote App 로드
- **실행 시점:** Remote App 배포 전 필수 게이트

```typescript
// Portal Integration Test
describe('Portal + Auth Remote App Integration', () => {
  it('Auth Remote App이 Shell에 정상적으로 마운트되어야 한다', async () => {
    await portalShell.loadRemoteApp('auth-app', '/auth');

    const mountPoint = document.getElementById('remote-app-container');
    expect(mountPoint.children.length).toBeGreaterThan(0);
  });

  it('Remote App에서 Portal SDK를 통해 알림을 표시할 수 있어야 한다', async () => {
    const notificationSpy = jest.spyOn(portalSDK, 'showNotification');

    await remoteApp.triggerAction('showSuccess');

    expect(notificationSpy).toHaveBeenCalledWith({
      type: 'success',
      title: expect.any(String),
      message: expect.any(String),
    });
  });

  it('Shell의 인증 상태 변경이 Remote App에 전파되어야 한다', async () => {
    const authChangeSpy = jest.fn();
    remoteApp.authSDK.onAuthStateChange(authChangeSpy);

    await portalShell.simulateLogout();

    expect(authChangeSpy).toHaveBeenCalledWith(null);
  });
});
```

### E2E Tests

- **도구:** Playwright
- **핵심 사용자 시나리오:**

**시나리오 1: 로그인 -> 대시보드 -> 프로젝트 탐색**

```typescript
test('사용자가 로그인하여 대시보드를 보고 프로젝트로 이동', async ({ page }) => {
  // 로그인
  await page.goto('/login');
  await page.fill('[data-testid="email"]', 'user@example.com');
  await page.fill('[data-testid="password"]', 'password');
  await page.click('[data-testid="login-button"]');

  // 대시보드 확인
  await expect(page.locator('[data-testid="dashboard"]')).toBeVisible();
  await expect(page.locator('[data-testid="recent-projects"]')).toBeVisible();

  // 프로젝트로 이동
  await page.click('[data-testid="project-link-proj-001"]');
  await expect(page).toHaveURL(/.*\/projects\/proj-001/);
});
```

**시나리오 2: 역할 생성 -> 권한 할당 -> 사용자 부여**

```typescript
test('관리자가 역할을 만들고 사용자에게 부여', async ({ page }) => {
  await loginAsAdmin(page);

  // 역할 생성
  await page.goto('/admin/roles');
  await page.click('[data-testid="create-role"]');
  await page.fill('[data-testid="role-name"]', 'Code Reviewer');
  await page.check('[data-testid="perm-CODE_READ"]');
  await page.check('[data-testid="perm-CODE_APPROVE"]');
  await page.click('[data-testid="save-role"]');

  await expect(page.locator('text=Code Reviewer')).toBeVisible();

  // 사용자에게 역할 부여
  await page.goto('/admin/users/user-002');
  await page.click('[data-testid="assign-role"]');
  await page.selectOption('[data-testid="role-select"]', 'Code Reviewer');
  await page.click('[data-testid="confirm-assign"]');

  await expect(page.locator('text=Code Reviewer')).toBeVisible();
});
```

**시나리오 3: 마스터 코드 생성 -> 승인 -> Lookup 검증**

```typescript
test('마스터 코드를 생성하고 승인 후 조회 가능 확인', async ({ page }) => {
  // 코드 생성 (일반 사용자)
  await loginAsUser(page);
  await page.goto('/mdm/codes');
  await page.click('[data-testid="create-code"]');
  await page.fill('[data-testid="group-code"]', 'TEST_GROUP');
  await page.fill('[data-testid="code-value"]', 'NEW_CODE');
  await page.fill('[data-testid="code-name"]', '신규 코드');
  await page.click('[data-testid="submit-code"]');

  await expect(page.locator('text=승인 대기')).toBeVisible();

  // 승인 (관리자)
  await loginAsAdmin(page);
  await page.goto('/mdm/approvals');
  await page.click('text=TEST_GROUP - NEW_CODE');
  await page.click('[data-testid="approve-button"]');

  // 조회 검증
  await page.goto('/mdm/lookup');
  await page.selectOption('[data-testid="group-select"]', 'TEST_GROUP');
  await expect(page.locator('text=신규 코드')).toBeVisible();
});
```

**시나리오 4: 권한 위임 -> 위임받은 사용자 접근 확인**

```typescript
test('권한 위임 후 위임받은 사용자가 접근 가능', async ({ page }) => {
  // 권한 위임
  await loginAsProjectAdmin(page);
  await page.goto('/admin/delegation');
  await page.click('[data-testid="create-delegation"]');
  await page.fill('[data-testid="delegatee"]', 'user-003');
  await page.selectOption('[data-testid="permission"]', 'CODE_APPROVE');
  await page.fill('[data-testid="expires-at"]', '2025-12-31');
  await page.click('[data-testid="confirm-delegation"]');

  // 위임받은 사용자로 접근 확인
  await loginAs(page, 'user-003');
  await page.goto('/mdm/approvals');
  await expect(page.locator('[data-testid="approve-button"]')).toBeEnabled();
});
```

### Permission Scenario Matrix

**Uncle Bob:** 권한 테스트는 매트릭스 형태로 체계적으로 검증합니다.

| 역할 | 리소스 | 액션 | 예상 결과 |
|------|--------|------|----------|
| SYSTEM_ADMIN | 역할 | 생성/수정/삭제 | ALLOW |
| SYSTEM_ADMIN | 마스터 코드 | 생성/수정/승인/삭제 | ALLOW |
| PROJECT_ADMIN | 역할 (프로젝트 내) | 생성/수정 | ALLOW |
| PROJECT_ADMIN | 역할 (다른 프로젝트) | 생성/수정 | DENY |
| MDM_ADMIN | 마스터 코드 | 생성/수정 | ALLOW |
| MDM_ADMIN | 마스터 코드 | 승인 (본인 생성) | DENY (자가 승인 금지) |
| MDM_VIEWER | 마스터 코드 | 조회 | ALLOW |
| MDM_VIEWER | 마스터 코드 | 생성/수정 | DENY |
| DELEGATEE | 위임된 권한 범위 | 위임된 액션 | ALLOW |
| DELEGATEE | 위임 범위 외 | 모든 액션 | DENY |
| 미인증 사용자 | 모든 리소스 | 모든 액션 | DENY (401) |

```typescript
// 매트릭스 기반 자동화 테스트
const permissionMatrix = [
  { role: 'SYSTEM_ADMIN', resource: 'role', action: 'create', expected: 'ALLOW' },
  { role: 'PROJECT_ADMIN', resource: 'role', action: 'create', projectScope: 'own', expected: 'ALLOW' },
  { role: 'PROJECT_ADMIN', resource: 'role', action: 'create', projectScope: 'other', expected: 'DENY' },
  { role: 'MDM_VIEWER', resource: 'master_code', action: 'create', expected: 'DENY' },
  // ... (전체 매트릭스)
];

describe('Permission Scenario Matrix', () => {
  permissionMatrix.forEach(({ role, resource, action, projectScope, expected }) => {
    it(`${role}가 ${resource}에 대해 ${action}을 시도하면 ${expected}`, async () => {
      const user = await createUserWithRole(role, projectScope);
      const result = await authService.checkPermission(user.id, resource, action);
      expect(result).toBe(expected === 'ALLOW');
    });
  });
});
```

### Performance Tests

**Uncle Bob:** 성능 테스트는 k6로 SLO 목표를 검증합니다.

```javascript
// k6 load test script
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up
    { duration: '5m', target: 100 },   // Steady state
    { duration: '2m', target: 200 },   // Peak
    { duration: '5m', target: 200 },   // Sustained peak
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],  // SLO
    http_req_failed: ['rate<0.001'],                  // 에러율 < 0.1%
  },
};

export default function () {
  // 마스터 코드 조회 (가장 빈번한 API)
  const lookupRes = http.get(`${BASE_URL}/api/master/lookup/PROJECT_STATUS`);
  check(lookupRes, { 'lookup status 200': (r) => r.status === 200 });

  // 권한 검증 (두 번째로 빈번)
  const authRes = http.get(`${BASE_URL}/api/auth/check-permission?perm=CODE_READ`);
  check(authRes, { 'auth check status 200': (r) => r.status === 200 });

  sleep(1);
}
```

### Security Tests

- **도구:** OWASP ZAP (자동화 스캔)
- **실행 시점:** Staging 배포 후 자동 실행
- **검증 항목:** OWASP Top 10, 인증 우회, 권한 상승, SQL Injection, XSS
- **결과:** CI 파이프라인에서 High 이상 발견 시 배포 차단

**Beck:** 테스트 전략이 매우 촘촘합니다. 이제 마지막으로, 이 모든 것을 개발자가 쉽게 사용할 수 있도록 하는 DX 설계로 넘어가겠습니다.

---

## 토의 5.7: 개발자 온보딩 / DX 설계

**Beck:** 개발자 경험(DX)은 플랫폼 성공의 핵심입니다. 아무리 좋은 아키텍처도 개발자가 사용하기 어려우면 의미가 없습니다. Karpathy, 이끌어 주시죠.

**Karpathy:** DX 설계의 핵심 원칙은 "30분 안에 Hello World를 띄울 수 있어야 한다"입니다. 그리고 가장 중요한 도구는 CLI입니다.

### portal-cli 도구

```bash
# 앱 스캐폴딩
portal-cli create-app --framework react --name my-app
# -> my-app/ 디렉토리 생성
# -> package.json, webpack.config.js, src/App.tsx, src/index.tsx 자동 생성
# -> Auth SDK, Master SDK, Portal SDK 자동 설치
# -> 개발 서버 설정 포함

# 로컬 개발 (Mock Shell, Auth, MDM 포함)
portal-cli dev
# -> Docker Compose로 의존 서비스 시작
# -> Mock Portal Shell 실행
# -> 내 앱을 Shell 안에서 hot-reload로 개발
# -> http://localhost:3000 에서 확인

# 테스트 실행
portal-cli test
# -> Unit Test + Contract Test + Integration Test 순차 실행
# -> 커버리지 리포트 생성

# Staging 배포
portal-cli deploy --env staging
# -> 빌드 -> 컨테이너 이미지 생성 -> Registry 푸시 -> Staging 배포
# -> Portal Integration Test 자동 실행

# 앱 등록 (App Registry에 등록)
portal-cli register
# -> manifest.json 기반으로 App Registry에 등록
# -> 접근 권한 설정, 메뉴 위치 설정
```

**Beck:** 각 명령어가 하나의 동작만 하고, 출력이 명확해야 합니다. 에러 시 해결 방법을 안내하는 것도 중요합니다.

**Karpathy:** 맞습니다. 에러 메시지는 다음 형식을 따릅니다.

```
ERROR: Auth Server에 연결할 수 없습니다.
CAUSE: Docker 컨테이너가 실행되지 않았을 수 있습니다.
FIX:   'portal-cli dev' 명령을 먼저 실행하세요.
DOCS:  https://portal-docs.example.com/troubleshooting#auth-connection
```

### 로컬 개발 환경

**Karpathy:** 로컬 개발 환경은 `docker-compose up` 한 줄로 전체 인프라를 시작할 수 있어야 합니다.

```yaml
# docker-compose.dev.yaml
version: '3.8'
services:
  # 인증 서버 (Mock 또는 경량 버전)
  auth-server:
    image: portal/auth-server:dev
    ports: ["8081:8080"]
    environment:
      - DB_URL=postgresql://postgres:dev@postgres:5432/auth_db
      - MOCK_SSO=true  # 로컬에서는 SSO 없이 직접 로그인

  # MDM Hub (경량 버전)
  mdm-hub:
    image: portal/mdm-hub:dev
    ports: ["8082:8080"]
    environment:
      - DB_URL=postgresql://postgres:dev@postgres:5432/mdm_db
      - REDIS_URL=redis://redis:6379

  # Portal Shell (개발 모드)
  portal-shell:
    image: portal/shell:dev
    ports: ["3000:3000"]
    environment:
      - AUTH_SERVER_URL=http://auth-server:8080
      - REMOTE_APP_DEV_MODE=true
    volumes:
      - ./remote-apps-config.json:/app/config/remote-apps.json

  # 인프라
  postgres:
    image: postgres:16
    ports: ["5432:5432"]
    environment:
      - POSTGRES_PASSWORD=dev
    volumes:
      - ./init-scripts:/docker-entrypoint-initdb.d  # Auth DB, MDM DB 초기화

  redis:
    image: redis:7
    ports: ["6379:6379"]
```

**Uncle Bob:** Hot reload가 핵심입니다. Remote App을 수정하면 브라우저가 즉시 반영되어야 합니다. Webpack Dev Server와 Module Federation의 hot reload를 활용합니다.

```
개발자 흐름:
1. git clone portal-starter-kit
2. cd my-app
3. portal-cli dev                    # 모든 의존 서비스 시작 (약 30초)
4. 코드 수정                          # Hot reload 즉시 반영
5. portal-cli test                   # 테스트 실행
6. portal-cli deploy --env staging   # 배포
```

**Nielsen:** 개발자 역시 사용자입니다. CLI의 UX도 중요합니다. 진행 상태를 표시하고, 색상으로 성공/실패를 구분하고, `--verbose` 모드를 제공하세요.

### 문서 구조

**Karpathy:** 문서는 다음 구조로 제공합니다.

| 문서 | 내용 | 대상 | 형식 |
|------|------|------|------|
| Getting Started | 30분 퀵스타트 가이드 | 신규 개발자 | 단계별 튜토리얼 |
| Architecture Overview | 전체 시스템 아키텍처 | 모든 개발자 | 다이어그램 + 설명 |
| SDK Reference | Auth/Master/Portal SDK API | 앱 개발자 | TypeDoc 자동 생성 |
| Cookbook | 시나리오별 레시피 | 앱 개발자 | 코드 중심 예제 |
| API Reference | 서비스 REST API | 백엔드 개발자 | OpenAPI (Swagger UI) |
| Troubleshooting | 자주 발생하는 문제와 해결법 | 모든 개발자 | FAQ 형식 |
| ADR | Architecture Decision Records | 시니어 개발자/아키텍트 | 결정 기록 |

**Beck:** Cookbook이 특히 중요합니다. "역할 기반 메뉴를 어떻게 숨기나요?", "마스터 코드 드롭다운을 어떻게 만드나요?" 같은 실제 질문에 답하는 문서죠.

```typescript
// Cookbook 예시: 권한 기반 조건부 렌더링
// recipes/conditional-rendering-by-permission.tsx

import { useAuth } from '@portal/auth-sdk/react';

function AdminPanel() {
  const { hasPermission } = useAuth();

  return (
    <div>
      <h1>프로젝트 관리</h1>

      {/* 역할 관리 섹션: ROLE_MANAGE 권한 필요 */}
      {hasPermission('ROLE_MANAGE') && (
        <section>
          <h2>역할 관리</h2>
          <RoleManagementTable />
        </section>
      )}

      {/* 코드 승인 섹션: CODE_APPROVE 권한 필요 */}
      {hasPermission('CODE_APPROVE') && (
        <section>
          <h2>승인 대기 코드</h2>
          <PendingApprovalList />
        </section>
      )}
    </div>
  );
}
```

```typescript
// Cookbook 예시: 마스터 코드 드롭다운 컴포넌트
// recipes/master-code-dropdown.tsx

import { useMasterCodes } from '@portal/master-sdk/react';

function StatusDropdown({ value, onChange }) {
  const { codes, isLoading } = useMasterCodes('PROJECT_STATUS', 'ko');

  if (isLoading) return <Spinner />;

  return (
    <select value={value} onChange={(e) => onChange(e.target.value)}>
      <option value="">선택하세요</option>
      {codes.map((code) => (
        <option key={code.codeValue} value={code.codeValue}>
          {code.codeName}
        </option>
      ))}
    </select>
  );
}
```

### Developer Portal (Self-Service)

**Karpathy:** 개발자가 스스로 관리할 수 있는 self-service 대시보드를 제공합니다.

| 기능 | 설명 |
|------|------|
| API Key 관리 | API Key 생성, 갱신, 폐기 |
| 앱 등록 | App Registry에 앱 등록 및 설정 |
| SDK 다운로드 | 각 언어별 SDK 패키지 및 예제 코드 |
| API 탐색 | Swagger UI로 API 직접 테스트 |
| 로그 뷰어 | 내 앱의 로그 실시간 조회 |
| 메트릭 대시보드 | 내 앱의 성능 지표 조회 |
| Sandbox 환경 | 격리된 테스트 환경 자동 생성 |

**Ng:** Developer Portal에 AI 기반 코드 어시스턴트를 통합하면 어떨까요. SDK 사용법에 대한 질문을 자연어로 하면 관련 문서와 예제 코드를 제시하는 방식입니다. RAG(Retrieval-Augmented Generation)을 사용해서 SDK 문서를 기반으로 답변을 생성할 수 있습니다.

**Karpathy:** 좋은 아이디어입니다. 초기에는 문서 검색 수준으로 시작하고, 점차 코드 생성까지 확장하면 됩니다.

**Beck:** DX에 대한 투자는 장기적으로 큰 보상을 줍니다. 개발자가 행복해야 좋은 앱이 나옵니다.

---

## 핵심 결정 사항 요약

| # | 결정 사항 | 선택 | 근거 |
|---|----------|------|------|
| 5.1-1 | 통신 패턴 | REST(동기) + Kafka(비동기) + WebSocket(실시간) | 용도별 최적 패턴 분리 |
| 5.1-2 | 네트워크 세그먼트 | Public / DMZ / Private 3단계 | 보안 심층 방어 |
| 5.1-3 | 비밀 관리 | HashiCorp Vault | 키 자동 교체, 감사 로그 |
| 5.2-1 | API Gateway | Kong | 플러그인 생태계, 검증된 솔루션 |
| 5.2-2 | Rate Limiting | 엔드포인트별 차등 (10/100/1000 req/min) | 보안 vs 사용성 균형 |
| 5.2-3 | BFF 패턴 | REST + BFF (GraphQL 추후 검토) | 초기 단순성 우선 |
| 5.2-4 | BFF 부분 실패 | Promise.allSettled + fallback | 하나의 서비스 장애가 전체 차단 방지 |
| 5.3-1 | Auth SDK | 동기 권한 체크 + 자동 토큰 갱신 | 개발자 편의, 내부 복잡성 캡슐화 |
| 5.3-2 | Master SDK | TTL 캐시 + 이벤트 기반 무효화 | 성능 + 일관성 균형 |
| 5.3-3 | Portal SDK | 이벤트 기반 Shell-App 통신 | 느슨한 결합, 독립 배포 |
| 5.3-4 | SDK 배포 | npm, Maven, PyPI, Go Modules | 다국어 지원 |
| 5.3-5 | SDK 버전 정책 | SemVer, 2개 메이저 버전 하위 호환 | 안정성과 진화의 균형 |
| 5.4-1 | 파이프라인 | 컴포넌트별 독립 파이프라인 | 독립적 배포 및 릴리스 |
| 5.4-2 | 통합 테스트 게이트 | Remote App은 Portal Integration Test 필수 | Shell 내 정상 동작 보장 |
| 5.4-3 | Contract Testing | Pact (Consumer-Driven) | 서비스 간 계약 보장 |
| 5.4-4 | 보안 스캐닝 | SonarQube + ZAP + Snyk + Trivy | 4단계 보안 검증 |
| 5.4-5 | 배포 전략 | Blue-Green (서비스) + Canary (프론트엔드) | 즉각 롤백 + 점진적 확인 |
| 5.4-6 | 롤백 시간 | 최대 5분 이내 | 운영 안정성 |
| 5.5-1 | 로그 포맷 | 구조화된 JSON, PII 자동 마스킹 | 검색 가능성 + 보안 |
| 5.5-2 | 로그 보관 | 30일 Hot / 90일 Warm / 1년 Cold | 비용 vs 필요성 균형 |
| 5.5-3 | 메트릭 | Golden Signals + 비즈니스 메트릭 | 인프라 + 비즈니스 가시성 |
| 5.5-4 | 분산 추적 | OpenTelemetry + W3C Trace Context | 업계 표준, 벤더 비종속 |
| 5.5-5 | 추적 샘플링 | 에러 100%, 정상 10% (프로덕션) | 비용 최적화 |
| 5.6-1 | 테스트 커버리지 | Unit 80%+, 전체 피라미드 구조 | 빠른 피드백 + 종합 검증 |
| 5.6-2 | E2E 도구 | Playwright | 크로스 브라우저, 안정성 |
| 5.6-3 | 성능 테스트 | k6 (p95 < 200ms, 에러율 < 0.1%) | SLO 기반 검증 |
| 5.6-4 | 권한 테스트 | 매트릭스 기반 자동화 | 체계적 커버리지 |
| 5.7-1 | CLI 도구 | portal-cli (create, dev, test, deploy, register) | 개발자 온보딩 간소화 |
| 5.7-2 | 로컬 개발 | Docker Compose (전체 스택, 30초 시작) | 환경 일관성 |
| 5.7-3 | 문서 구조 | 7개 카테고리 (Getting Started ~ ADR) | 대상별 맞춤 문서 |
| 5.7-4 | Developer Portal | Self-service 대시보드 | 자율성 + 생산성 |

---

> **다음 파트 예고 (Part 6):** 비기능 요구사항 및 운영 설계 - 성능, 확장성, 가용성, 재해 복구, 데이터 마이그레이션, 운영 절차를 상세히 논의합니다.
