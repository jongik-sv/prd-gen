# Part 7: TRD 문서 작성 - 대화형 토론 기록

> 본 문서는 포털 플랫폼의 기술 참조 문서(TRD) 작성을 위한 전문가 브레인스토밍 토론을 기록한 것입니다.

## 참여 전문가

| 분야 | 인물 |
|------|------|
| AI | Andrew Ng + Andrej Karpathy |
| DB | Martin Kleppmann |
| 아키텍처 | Martin Fowler |
| SW | Robert C. Martin (Uncle Bob) |
| PM | Kent Beck (사회자) |
| UX/보안 | Jakob Nielsen + Bruce Schneier |
| 문서화 | Donald Knuth (스타일) |

---

## 토의 1: 시스템 아키텍처 다이어그램 (7.1)

**Kent Beck:** 좋습니다, 여러분. TRD 문서의 첫 번째 주제로 시스템 아키텍처 전체 조감도를 그려보겠습니다. Martin Fowler, 전체 시스템의 레이어 구성을 먼저 발표해 주시겠습니까?

**Martin Fowler:** 네, 감사합니다. 우리 포털 플랫폼은 크게 다섯 개의 레이어로 구성됩니다. 각 레이어는 명확한 책임 경계를 가지며, 상위 레이어는 하위 레이어에만 의존하는 단방향 의존성 원칙을 따릅니다. 전체 구조를 도식화하면 다음과 같습니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Layer                              │
│  Browser ─── Shell App ─── Remote Apps (MF)                 │
├─────────────────────────────────────────────────────────────┤
│                    Edge Layer                                │
│  CDN (CloudFront) ── API Gateway (Kong) ── Load Balancer    │
├─────────────────────────────────────────────────────────────┤
│                 Application Layer                            │
│  Portal BFF ── Auth Service ── MDM Service ── Notification  │
├─────────────────────────────────────────────────────────────┤
│                    Data Layer                                │
│  PostgreSQL ── Redis ── Elasticsearch ── Kafka              │
├─────────────────────────────────────────────────────────────┤
│                 Cross-cutting Concerns                       │
│  OpenTelemetry ── Grafana ── Vault ── CI/CD                │
└─────────────────────────────────────────────────────────────┘
```

**Martin Fowler:** Client Layer에서는 브라우저가 Shell App을 로드하고, Shell App이 Module Federation을 통해 Remote App들을 동적으로 불러옵니다. 여기서 중요한 것은 Shell이 오케스트레이터 역할을 한다는 점입니다. 각 Remote App은 독립적으로 빌드되고 배포되지만, 런타임에서는 Shell의 컨텍스트 안에서 통합됩니다.

**Kent Beck:** 각 레이어 간의 통신 패턴에 대해 좀 더 구체적으로 설명해 주시겠습니까?

**Martin Fowler:** 물론입니다. 통신 패턴은 세 가지로 나뉩니다.

첫째, **동기 통신(Synchronous)**은 REST API를 사용합니다. Client에서 Edge Layer를 거쳐 Application Layer로 들어오는 모든 요청이 여기에 해당합니다. BFF가 클라이언트의 단일 진입점 역할을 하며, 내부적으로 Auth Service나 MDM Service를 호출할 때도 REST를 사용합니다.

```
[Browser] → HTTPS → [CloudFront] → HTTPS → [Kong API Gateway]
    → HTTP/2 → [Portal BFF]
        → gRPC/REST → [Auth Service]
        → gRPC/REST → [MDM Service]
```

둘째, **비동기 통신(Asynchronous)**은 Kafka를 사용합니다. 서비스 간 이벤트 전파가 필요한 경우입니다. 예를 들어, 권한 변경이 발생하면 Auth Service가 Kafka에 이벤트를 발행하고, 관련 서비스들이 이를 구독하여 캐시를 갱신합니다.

```
[Auth Service] → Kafka Topic: auth.permission.changed
    → [Portal BFF] (캐시 무효화)
    → [Notification Service] (알림 발송)
    → [MDM Service] (연관 데이터 갱신)
```

셋째, **실시간 통신(Real-time)**은 WebSocket을 사용합니다. 알림 서비스가 사용자에게 실시간 알림을 전달할 때 사용합니다.

```
[Notification Service] → WebSocket → [Kong] → [Browser]
    - 승인 요청 알림
    - 권한 변경 통지
    - 시스템 공지사항
```

**Andrej Karpathy:** 한 가지 질문이 있습니다. BFF에서 여러 백엔드 서비스를 호출할 때 장애 전파가 우려됩니다. Circuit Breaker 패턴은 어떻게 적용합니까?

**Martin Fowler:** 아주 좋은 지적입니다. BFF에서 각 서비스 호출에는 Circuit Breaker를 적용합니다. NestJS 환경에서는 `@nestjs/terminus`와 함께 커스텀 Circuit Breaker 데코레이터를 구현합니다. 상태는 세 가지입니다:

```
CLOSED (정상) → 실패율 50% 초과 → OPEN (차단)
OPEN → 30초 후 → HALF-OPEN (시험)
HALF-OPEN → 성공 → CLOSED / 실패 → OPEN
```

각 서비스 호출에 대해 독립적인 Circuit Breaker 인스턴스를 유지하므로, Auth Service가 장애를 겪더라도 MDM Service 호출에는 영향을 주지 않습니다.

**Bruce Schneier:** 네트워크 보안 관점에서 말씀드리겠습니다. 아키텍처의 네트워크 세그먼트를 명확히 정의해야 합니다. 저는 세 개의 보안 영역(Security Zone)을 제안합니다.

```
┌──────────────────────────────────────────────────────────────────┐
│  Public Zone (인터넷 접근 가능)                                    │
│  ┌──────────┐  ┌──────────────┐                                  │
│  │   CDN    │  │ Load Balancer│                                  │
│  │CloudFront│  │   (NLB/ALB)  │                                  │
│  └────┬─────┘  └──────┬───────┘                                  │
│───────┼────────────────┼─────────────────────────────────────────│
│  DMZ (제한적 접근)       │                                         │
│  ┌─────────────────────┴──────┐                                  │
│  │      API Gateway (Kong)    │  ← WAF, Rate Limit, JWT 검증     │
│  └─────────────┬──────────────┘                                  │
│────────────────┼─────────────────────────────────────────────────│
│  Private Zone (내부 네트워크만 접근)                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │Portal BFF│  │Auth Svc  │  │MDM Svc   │  │Notification Svc  │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────────────┘ │
│───────┼──────────────┼────────────┼──────────────┼───────────────│
│  Data Zone (최소 권한 접근)                                        │
│  ┌──────────┐  ┌─────┐  ┌─────────────┐  ┌─────┐               │
│  │PostgreSQL│  │Redis│  │Elasticsearch│  │Kafka│               │
│  └──────────┘  └─────┘  └─────────────┘  └─────┘               │
└──────────────────────────────────────────────────────────────────┘
```

**Bruce Schneier:** 핵심 원칙은 다음과 같습니다. Public Zone에서는 절대로 Private Zone에 직접 접근할 수 없습니다. 반드시 DMZ의 API Gateway를 통과해야 합니다. Gateway에서는 WAF(Web Application Firewall)가 OWASP Top 10 공격을 필터링하고, Rate Limiting으로 DDoS를 완화하며, JWT 토큰의 유효성을 1차 검증합니다.

Private Zone의 서비스들은 Data Zone에 접근할 때도 최소 권한 원칙을 따릅니다. 예를 들어, Notification Service는 PostgreSQL에 직접 접근하지 못하고, 필요한 데이터는 Kafka 이벤트로 수신하거나 BFF를 통해 요청합니다.

**Martin Kleppmann:** 보안 영역과 더불어 장애 도메인(Fault Domain) 격리에 대해서도 논의가 필요합니다. 데이터 레이어에서 PostgreSQL이 다운되더라도 Redis 캐시에 저장된 세션 정보로 일정 시간 동안 서비스를 유지할 수 있어야 합니다. 이를 위해 각 데이터 저장소는 독립적인 장애 도메인으로 운영됩니다.

```
Fault Domain 1: PostgreSQL (Primary + Read Replica)
  → 장애 시: 읽기는 Replica로 전환, 쓰기는 큐잉 후 재시도

Fault Domain 2: Redis Cluster (3 노드)
  → 장애 시: 로컬 캐시 폴백, 세션은 JWT 자체 검증으로 전환

Fault Domain 3: Elasticsearch
  → 장애 시: 검색 기능 비활성화, 기본 CRUD는 PostgreSQL로 처리

Fault Domain 4: Kafka Cluster (3 브로커)
  → 장애 시: 동기 호출로 폴백, Dead Letter Queue에 실패 이벤트 저장
```

**Kent Beck:** 좋습니다. 그러면 배포 단위(Deployment Unit)에 대해 정리해 봅시다. 독립적으로 배포 가능한 단위가 무엇인지 표로 정리해 주시겠습니까?

**Martin Fowler:** 네, 각 배포 단위와 그 특성을 표로 정리하겠습니다.

| 배포 단위 | 기술 스택 | 배포 대상 | 스케일링 전략 | 의존성 |
|-----------|-----------|-----------|--------------|--------|
| Shell App | React + Vite | CloudFront (S3) | CDN 캐싱 | 없음 (정적 자산) |
| Auth Admin App | React + Vite (MF Remote) | CloudFront (S3) | CDN 캐싱 | Shell App (런타임) |
| Master Admin App | React + Vite (MF Remote) | CloudFront (S3) | CDN 캐싱 | Shell App (런타임) |
| Portal BFF | NestJS | K8s Deployment | HPA (CPU 70%) | Auth Service, MDM Service |
| Auth Service | NestJS | K8s Deployment | HPA (CPU 70%) | PostgreSQL, Redis, Kafka |
| MDM Service | NestJS | K8s Deployment | HPA (CPU 70%) | PostgreSQL, Redis, Kafka |
| Notification Service | NestJS | K8s Deployment | HPA (메시지 큐 깊이) | Kafka, Redis |
| PostgreSQL | PostgreSQL 15 | RDS / K8s StatefulSet | Read Replica | 없음 |
| Redis | Redis 7 | ElastiCache / K8s | Cluster 모드 | 없음 |
| Kafka | Kafka 3.x | MSK / K8s StatefulSet | 파티션 확장 | 없음 |
| Elasticsearch | ES 8.x | K8s StatefulSet | 노드 추가 | 없음 |

**Martin Fowler:** 여기서 핵심은 프론트엔드 앱들이 완전히 독립적으로 배포된다는 것입니다. Shell App을 배포하지 않아도 Auth Admin App만 단독으로 새 버전을 배포할 수 있습니다. 이것이 Module Federation의 핵심 가치입니다.

**Robert C. Martin:** 배포 독립성을 보장하려면 각 서비스의 API 계약이 하위 호환성을 유지해야 합니다. 저는 API 버저닝을 반드시 도입할 것을 권합니다. URL 경로에 `/api/v1/`, `/api/v2/` 형태로 명시하고, 최소 두 개 버전을 동시에 지원해야 합니다.

**Bruce Schneier:** 보안 경계에 대해 한 가지 더 강조하겠습니다. 각 영역 간 통신에는 반드시 mTLS(Mutual TLS)를 적용해야 합니다. 특히 Private Zone 내부에서도 서비스 간 통신은 서비스 메시(Service Mesh)를 통해 암호화됩니다. Vault가 인증서 수명주기를 자동으로 관리하고, 24시간마다 인증서를 갱신합니다.

```
[Kong] ──mTLS──→ [Portal BFF] ──mTLS──→ [Auth Service]
                                  ──mTLS──→ [MDM Service]

인증서 관리:
  Vault (PKI Secret Engine)
    → 자동 발급: 서비스 시작 시
    → 자동 갱신: TTL 24h, 만료 1시간 전 갱신
    → 자동 폐기: 서비스 종료 시
```

**Andrew Ng:** 관찰 가능성(Observability) 측면에서 Cross-cutting Concerns 레이어를 보완하고 싶습니다. OpenTelemetry를 통한 분산 추적에서는 모든 요청에 Trace ID를 부여하고, 이 ID가 Client부터 Data Layer까지 전파되어야 합니다. 장애 발생 시 단일 Trace ID로 전체 요청 흐름을 재구성할 수 있어야 합니다.

```
Trace 전파 경로:
  Browser (X-Trace-ID 헤더)
    → Kong (헤더 전달 + 접근 로그)
      → BFF (Span 생성: bff.request)
        → Auth Service (Span 생성: auth.verify)
        → MDM Service (Span 생성: mdm.query)
          → PostgreSQL (Span 생성: db.query)
    → Grafana Tempo (Trace 수집 및 시각화)
```

**Kent Beck:** 훌륭합니다. 이 아키텍처는 각 레이어의 책임이 명확하고, 장애 격리와 보안 경계가 잘 정의되어 있습니다. 정리하겠습니다.

> **합의:** 시스템은 Client, Edge, Application, Data, Cross-cutting의 5개 레이어로 구성한다. 동기 통신은 REST/gRPC, 비동기는 Kafka, 실시간은 WebSocket을 사용한다. 네트워크는 Public Zone, DMZ, Private Zone, Data Zone의 4개 보안 영역으로 분리하며, 영역 간 통신에는 mTLS를 적용한다. 각 데이터 저장소는 독립적 장애 도메인으로 운영하고, 배포 단위는 프론트엔드 3개, 백엔드 4개, 인프라 4개로 총 11개이며 각각 독립 배포가 가능하다. 모든 요청에 Trace ID를 전파하여 End-to-End 관찰 가능성을 확보한다.

---

## 토의 2: 디렉토리 구조 명세 (7.2)

**Kent Beck:** 다음 주제로 넘어가겠습니다. Uncle Bob, 모노레포의 전체 디렉토리 구조를 제안해 주시겠습니까? 개발자가 이 프로젝트를 처음 열었을 때 어떤 구조를 보게 되는지가 중요합니다.

**Robert C. Martin:** 감사합니다. 저는 Clean Architecture의 원칙을 모노레포 수준에서도 적용하는 것이 중요하다고 생각합니다. 최상위 디렉토리는 관심사의 레벨에 따라 분리됩니다. `apps/`는 사용자 인터페이스, `packages/`는 공유 라이브러리, `services/`는 백엔드 비즈니스 로직, `infra/`는 인프라스트럭처입니다. 전체 구조는 다음과 같습니다.

```
portal-platform/
├── apps/
│   ├── shell/                    # Portal Shell (React + Vite)
│   │   ├── src/
│   │   │   ├── components/       # Shell-specific components
│   │   │   ├── layouts/          # Header, Sidebar, Footer
│   │   │   ├── pages/            # Shell pages (Dashboard, 404, 403)
│   │   │   ├── router/           # Route registry, guards
│   │   │   ├── store/            # Shell state (Zustand)
│   │   │   ├── services/         # API clients
│   │   │   ├── hooks/            # Custom hooks
│   │   │   └── adapters/         # MF/SPA/iframe adapters
│   │   ├── public/
│   │   ├── tests/
│   │   └── vite.config.ts
│   ├── auth-admin/               # 권한관리 앱 (React + Vite)
│   │   ├── src/
│   │   │   ├── features/
│   │   │   │   ├── roles/        # 역할 관리
│   │   │   │   ├── permissions/  # 권한 매트릭스
│   │   │   │   ├── delegations/  # 위임 관리
│   │   │   │   ├── approvals/    # 승인 워크플로우
│   │   │   │   └── audit/        # 감사 로그
│   │   │   ├── components/
│   │   │   └── hooks/
│   │   └── ...
│   └── master-admin/             # 마스터관리 앱 (React + Vite)
│       ├── src/
│       │   ├── features/
│       │   │   ├── code-groups/  # 코드 그룹 관리
│       │   │   ├── codes/        # 코드 CRUD
│       │   │   ├── bulk/         # 벌크 처리
│       │   │   ├── governance/   # 변경 거버넌스
│       │   │   └── quality/      # 품질 대시보드
│       │   └── ...
│       └── ...
├── packages/
│   ├── design-tokens/            # CSS Variables, JSON tokens
│   ├── ui/                       # 공통 UI 컴포넌트 (React)
│   ├── auth-sdk/                 # Auth SDK (TypeScript)
│   ├── master-sdk/               # Master SDK (TypeScript)
│   ├── portal-sdk/               # Portal SDK (TypeScript)
│   └── shared-types/             # 공유 TypeScript 타입
├── services/
│   ├── auth-service/             # NestJS 인증/인가 서비스
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   │   ├── auth/         # 인증 (login, logout, refresh)
│   │   │   │   ├── users/        # 사용자 관리
│   │   │   │   ├── roles/        # 역할 관리
│   │   │   │   ├── permissions/  # 권한 관리
│   │   │   │   ├── delegations/  # 위임
│   │   │   │   ├── approvals/    # 승인
│   │   │   │   └── audit/        # 감사 로그
│   │   │   ├── guards/           # Auth guards
│   │   │   ├── decorators/       # Custom decorators
│   │   │   ├── entities/         # TypeORM entities
│   │   │   └── common/           # Shared utilities
│   │   ├── test/
│   │   └── nest-cli.json
│   ├── mdm-service/              # NestJS MDM 서비스
│   ├── portal-bff/               # NestJS Portal BFF
│   └── notification-service/     # NestJS 알림 서비스
├── infra/
│   ├── docker/
│   ├── k8s/
│   ├── terraform/
│   └── scripts/
├── docs/                         # ADR, API docs, guides
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
└── .github/workflows/
```

**Robert C. Martin:** 이 구조에서 제가 특히 강조하고 싶은 것은 각 서비스 내부의 Clean Architecture 적용입니다. `services/auth-service/`를 예로 들겠습니다.

```
services/auth-service/src/
├── modules/                  # 기능 모듈 (Use Cases 레이어)
│   └── auth/
│       ├── auth.module.ts        # NestJS 모듈 정의
│       ├── auth.controller.ts    # 진입점 (Interface Adapter)
│       ├── auth.service.ts       # 비즈니스 로직 (Use Case)
│       ├── auth.repository.ts    # 데이터 접근 (Interface Adapter)
│       ├── dto/                  # Data Transfer Objects
│       │   ├── login.dto.ts
│       │   ├── token-refresh.dto.ts
│       │   └── logout.dto.ts
│       └── __tests__/            # 모듈별 단위 테스트
│           ├── auth.service.spec.ts
│           └── auth.controller.spec.ts
├── guards/                   # 횡단 관심사 (Framework 레이어)
│   ├── jwt-auth.guard.ts
│   ├── roles.guard.ts
│   └── permission.guard.ts
├── decorators/               # 커스텀 데코레이터
│   ├── current-user.decorator.ts
│   ├── roles.decorator.ts
│   └── permissions.decorator.ts
├── entities/                 # 엔티티 (Enterprise Business Rules)
│   ├── user.entity.ts
│   ├── role.entity.ts
│   ├── permission.entity.ts
│   └── delegation.entity.ts
├── common/                   # 공유 유틸리티
│   ├── filters/              # Exception filters
│   ├── interceptors/         # Logging, transform interceptors
│   ├── pipes/                # Validation pipes
│   └── constants/            # 상수 정의
└── main.ts                   # 애플리케이션 엔트리포인트
```

**Robert C. Martin:** 여기서 핵심 원칙은 **의존성 방향**입니다. Controller는 Service에 의존하고, Service는 Repository 인터페이스에 의존하며, 실제 Repository 구현체는 Entity에 의존합니다. 어떤 경우에도 Entity가 외부 프레임워크에 의존해서는 안 됩니다. TypeORM 데코레이터를 Entity에 사용하는 것은 타협이지만, 이 경우에도 비즈니스 로직은 Entity 메서드 안에 순수 함수로 구현합니다.

```typescript
// entities/user.entity.ts - 비즈니스 규칙은 Entity 내부에
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  loginId: string;

  @Column()
  passwordHash: string;

  @Column({ type: 'timestamp', nullable: true })
  lastLoginAt: Date | null;

  @Column({ default: 0 })
  failedLoginCount: number;

  // 순수 비즈니스 로직 - 프레임워크 의존성 없음
  isAccountLocked(): boolean {
    return this.failedLoginCount >= 5;
  }

  canDelegate(targetRole: Role): boolean {
    return this.roles.some(r => r.level > targetRole.level);
  }

  recordLoginFailure(): void {
    this.failedLoginCount += 1;
  }

  resetLoginFailure(): void {
    this.failedLoginCount = 0;
    this.lastLoginAt = new Date();
  }
}
```

**Martin Fowler:** Uncle Bob의 구조에 동의합니다. 제가 보충하고 싶은 것은 `apps/`, `packages/`, `services/`의 경계 원칙입니다.

첫째, `apps/`는 **배포 가능한 프론트엔드 단위**입니다. 각 앱은 자체 `vite.config.ts`와 `package.json`을 가지며, 독립적으로 빌드됩니다. 앱 간 직접 import는 금지되며, 반드시 `packages/`를 통해서만 코드를 공유합니다.

둘째, `packages/`는 **순수 라이브러리**입니다. 런타임 의존성이 최소화되어야 하고, 앱이나 서비스에 의존해서는 안 됩니다. 의존성 방향은 항상 `apps/ → packages/` 또는 `services/ → packages/`입니다.

```
의존성 규칙:

  apps/shell ──────→ packages/ui
                 ──→ packages/portal-sdk
                 ──→ packages/auth-sdk
                 ──→ packages/design-tokens
                 ──→ packages/shared-types

  apps/auth-admin ──→ packages/ui
                  ──→ packages/auth-sdk
                  ──→ packages/design-tokens
                  ──→ packages/shared-types

  services/auth-service ──→ packages/shared-types

  ✗ 금지: apps/shell ──→ apps/auth-admin (앱 간 직접 의존)
  ✗ 금지: packages/ui ──→ apps/shell (역방향 의존)
  ✗ 금지: services/auth-service ──→ packages/ui (백엔드→프론트엔드)
```

셋째, `services/`는 **배포 가능한 백엔드 단위**입니다. 서비스 간에도 직접 코드 import는 불가하며, 네트워크 호출(REST/gRPC/Kafka)로만 통신합니다. 공유가 필요한 타입은 `packages/shared-types`에 정의합니다.

**Andrej Karpathy:** 개발자 경험(DX) 관점에서 한 가지 말씀드리겠습니다. 새로운 개발자가 이 모노레포를 처음 클론했을 때의 경험을 시뮬레이션해 봤습니다.

```bash
# 1. 저장소 클론
git clone https://github.com/org/portal-platform.git
cd portal-platform

# 2. 의존성 설치 (pnpm workspace가 모든 패키지를 한 번에 설치)
pnpm install

# 3. 전체 빌드 (Turborepo가 의존성 순서대로 빌드)
pnpm turbo build

# 4. 개발 서버 시작 (관심 있는 앱만)
pnpm turbo dev --filter=shell    # Shell만 개발
pnpm turbo dev --filter=shell... # Shell + 의존하는 모든 패키지
```

**Andrej Karpathy:** 중요한 것은 **진입 장벽을 낮추는 것**입니다. 새 개발자는 최상위 디렉토리만 봐도 전체 시스템의 구조를 직관적으로 이해할 수 있어야 합니다. `apps/`에 사용자가 보는 화면이 있고, `packages/`에 공유 코드가 있고, `services/`에 백엔드가 있고, `infra/`에 인프라 코드가 있다. 이 네 개의 디렉토리명만으로도 아키텍처의 큰 그림이 그려집니다.

또한 각 `feature/` 디렉토리 안에서는 관련 코드가 Co-location 원칙에 따라 한 곳에 모여 있습니다. 역할 관리 기능을 개발한다면 `apps/auth-admin/src/features/roles/` 한 폴더만 보면 됩니다. 컴포넌트, 훅, 타입, 테스트가 모두 여기에 있습니다.

```
apps/auth-admin/src/features/roles/
├── components/
│   ├── RoleList.tsx
│   ├── RoleDetail.tsx
│   ├── RoleForm.tsx
│   └── RoleTreeView.tsx
├── hooks/
│   ├── useRoles.ts
│   ├── useRoleDetail.ts
│   └── useRoleMutation.ts
├── types/
│   └── role.types.ts
├── api/
│   └── role.api.ts
├── __tests__/
│   ├── RoleList.test.tsx
│   └── useRoles.test.ts
└── index.ts                  # Public API (barrel export)
```

**Kent Beck:** Turborepo 파이프라인 설정에 대해서도 논의해야 합니다. 빌드 순서와 캐싱 전략이 개발 생산성에 직접적인 영향을 미치기 때문입니다.

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "build/**"],
      "cache": true
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": ["coverage/**"],
      "cache": true
    },
    "lint": {
      "outputs": [],
      "cache": true
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "outputs": [],
      "cache": true
    }
  }
}
```

**Kent Beck:** `"dependsOn": ["^build"]`에서 `^` 기호는 해당 패키지의 의존성이 먼저 빌드되어야 함을 의미합니다. 예를 들어 `apps/shell`이 `packages/ui`에 의존하면, `packages/ui`가 먼저 빌드된 후에 `apps/shell`이 빌드됩니다. Turborepo는 이 의존성 그래프를 분석하여 최대한 병렬로 실행합니다.

```
빌드 순서 (Turborepo가 자동 결정):

Phase 1 (병렬): packages/design-tokens
                 packages/shared-types

Phase 2 (병렬): packages/ui (design-tokens에 의존)
                 packages/auth-sdk (shared-types에 의존)
                 packages/master-sdk (shared-types에 의존)
                 packages/portal-sdk (shared-types에 의존)

Phase 3 (병렬): apps/shell (ui, portal-sdk, auth-sdk에 의존)
                 apps/auth-admin (ui, auth-sdk에 의존)
                 apps/master-admin (ui, master-sdk에 의존)
                 services/auth-service (shared-types에 의존)
                 services/mdm-service (shared-types에 의존)
                 services/portal-bff (shared-types에 의존)
                 services/notification-service (shared-types에 의존)
```

**Kent Beck:** 캐싱 전략도 중요합니다. Turborepo는 입력 파일의 해시를 기반으로 캐시를 결정합니다. `packages/shared-types`가 변경되지 않았다면, 이에 의존하는 모든 패키지의 빌드 결과를 캐시에서 가져옵니다. CI 환경에서는 Remote Cache를 활성화하여 팀원 간 빌드 캐시를 공유할 수 있습니다.

**Martin Kleppmann:** 데이터 마이그레이션 스크립트의 위치에 대해 제안하겠습니다. 마이그레이션은 서비스와 밀접하게 결합되어 있으므로, 각 서비스 디렉토리 안에 배치하는 것이 원칙입니다.

```
services/auth-service/
├── src/
│   └── ...
├── migrations/                       # TypeORM 마이그레이션
│   ├── 1700000000000-CreateUserTable.ts
│   ├── 1700000001000-CreateRoleTable.ts
│   ├── 1700000002000-CreatePermissionTable.ts
│   ├── 1700000003000-AddDelegationTable.ts
│   └── 1700000004000-AddAuditLogTable.ts
├── seeds/                            # 초기 데이터 시딩
│   ├── 001-default-roles.seed.ts
│   ├── 002-system-permissions.seed.ts
│   └── 003-admin-user.seed.ts
├── test/
│   └── ...
└── nest-cli.json
```

**Martin Kleppmann:** 마이그레이션이 서비스 디렉토리 안에 있는 이유는 세 가지입니다.

첫째, **서비스 자율성**입니다. 각 서비스는 자신의 데이터 스키마를 소유합니다. Auth Service의 마이그레이션은 Auth Service 팀이 관리하며, 다른 서비스의 마이그레이션에 영향을 받지 않습니다.

둘째, **배포 원자성**입니다. 서비스 코드와 마이그레이션이 같은 디렉토리에 있으므로, 동일한 PR에서 코드 변경과 스키마 변경을 함께 리뷰할 수 있습니다. 코드와 스키마의 불일치를 방지할 수 있습니다.

셋째, **CI/CD 통합**입니다. 배포 파이프라인에서 서비스를 배포하기 전에 해당 서비스의 마이그레이션을 자동으로 실행할 수 있습니다.

```yaml
# .github/workflows/deploy-auth-service.yml 내부
steps:
  - name: Run migrations
    run: |
      cd services/auth-service
      pnpm typeorm migration:run
  - name: Run seeds (staging only)
    if: github.ref == 'refs/heads/staging'
    run: |
      cd services/auth-service
      pnpm seed:run
  - name: Deploy service
    run: |
      # Kubernetes rolling update
```

**Martin Kleppmann:** 단, 서비스 간 공유되는 스키마가 필요한 경우(예: 공통 감사 로그 테이블 형식)는 `packages/shared-types`에 인터페이스만 정의하고, 실제 테이블 생성은 각 서비스가 독립적으로 수행합니다. 데이터베이스 수준의 공유는 최소화하는 것이 마이크로서비스의 원칙입니다.

**Robert C. Martin:** Kleppmann의 의견에 전적으로 동의합니다. 보충하자면, `infra/` 디렉토리에는 서비스와 무관한 인프라 수준의 스크립트만 배치합니다.

```
infra/
├── docker/
│   ├── docker-compose.yml            # 로컬 개발 환경
│   ├── docker-compose.test.yml       # 테스트 환경
│   └── Dockerfile.service            # 서비스 공통 Dockerfile
├── k8s/
│   ├── base/                         # Kustomize base
│   │   ├── namespace.yaml
│   │   ├── network-policy.yaml       # 네트워크 정책 (보안 영역)
│   │   └── resource-quotas.yaml
│   ├── overlays/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── production/
│   └── charts/                       # Helm charts (외부 의존성)
│       ├── postgresql/
│       ├── redis/
│       ├── kafka/
│       └── elasticsearch/
├── terraform/
│   ├── modules/
│   │   ├── vpc/
│   │   ├── eks/
│   │   ├── rds/
│   │   └── elasticache/
│   ├── environments/
│   │   ├── dev/
│   │   ├── staging/
│   │   └── production/
│   └── backend.tf
└── scripts/
    ├── setup-local.sh                # 로컬 개발 환경 초기화
    ├── seed-all.sh                   # 모든 서비스 시딩 실행
    └── generate-api-docs.sh          # API 문서 생성
```

**Jakob Nielsen:** UX 관점에서 개발자도 사용자라는 점을 강조하고 싶습니다. `docs/` 디렉토리의 구조도 중요합니다. 새로운 개발자가 가장 먼저 찾는 것은 "어디서부터 시작하면 되는가"입니다.

```
docs/
├── getting-started.md                # 가장 먼저 읽는 문서
├── architecture/
│   ├── adr/                          # Architecture Decision Records
│   │   ├── 001-monorepo-structure.md
│   │   ├── 002-module-federation.md
│   │   ├── 003-auth-strategy.md
│   │   └── template.md
│   ├── diagrams/                     # 아키텍처 다이어그램 원본
│   └── overview.md
├── api/
│   ├── auth-service/                 # OpenAPI spec
│   ├── mdm-service/
│   └── portal-bff/
├── guides/
│   ├── adding-new-feature.md
│   ├── adding-new-service.md
│   ├── adding-new-remote-app.md
│   └── deployment-guide.md
└── conventions/
    ├── coding-standards.md
    ├── git-workflow.md
    └── naming-conventions.md
```

**Kent Beck:** 모두 훌륭한 의견을 주셨습니다. 디렉토리 구조에 대해 합의를 정리하겠습니다.

> **합의:** 모노레포는 `apps/`(프론트엔드 배포 단위), `packages/`(공유 라이브러리), `services/`(백엔드 배포 단위), `infra/`(인프라 코드), `docs/`(문서)의 5개 최상위 디렉토리로 구성한다. 의존성 방향은 항상 apps/services에서 packages로의 단방향이며, 앱 간 또는 서비스 간 직접 import는 금지한다. 각 서비스 내부는 Clean Architecture 원칙에 따라 modules/guards/entities/common으로 분리하고, 프론트엔드 앱은 feature 기반 Co-location 원칙을 따른다. 데이터 마이그레이션 스크립트는 각 서비스 디렉토리 내에 `migrations/`와 `seeds/`로 배치한다. Turborepo가 의존성 그래프 기반으로 빌드 순서를 결정하며, Remote Cache를 활용하여 빌드 성능을 최적화한다.

---
## 토의 3: TypeScript 인터페이스/타입 정의 (7.3)

**Name: Kent Beck (사회자)**
세 번째 주제로 넘어가겠습니다. TypeScript 인터페이스와 타입 정의를 체계적으로 설계해야 합니다. 이 부분은 Uncle Bob이 리드해 주시죠. 타입 시스템이 곧 문서이자 계약이니까요.

**Name: Robert C. Martin (Uncle Bob)**
감사합니다. 클린 아키텍처 관점에서 타입 정의는 시스템의 뼈대입니다. 인터페이스를 잘 정의하면 구현은 자연스럽게 따라옵니다. 먼저 모든 API에서 공통으로 사용할 기본 타입부터 정의하겠습니다.

### 7.3.1 공통 타입 (Common Types)

**Name: Robert C. Martin (Uncle Bob)**
모든 API 응답은 하나의 표준 형식을 따라야 합니다. 성공이든 실패든 일관된 구조를 클라이언트에게 제공하는 것이 핵심입니다.

```typescript
/** 표준 API 응답 래퍼 - 모든 엔드포인트가 이 형식을 반환 */
interface ApiResponse<T> {
  /** 요청 처리 결과 상태 */
  status: 'success' | 'error';
  /** 응답 데이터 (성공 시) */
  data: T;
  /** 에러 정보 (실패 시) */
  error?: ApiError;
  /** 응답 메타데이터 */
  meta?: ResponseMeta;
}

/** API 에러 상세 정보 */
interface ApiError {
  /** 에러 코드 (예: AUTH_001, MASTER_003) */
  code: string;
  /** 사람이 읽을 수 있는 에러 메시지 */
  message: string;
  /** 추가 에러 상세 (필드별 검증 오류 등) */
  details?: Record<string, any>;
}

/** 응답 메타데이터 - 추적 및 디버깅 용도 */
interface ResponseMeta {
  /** 요청 고유 식별자 (UUID v4) */
  requestId: string;
  /** ISO 8601 형식 타임스탬프 */
  timestamp: string;
}

/** 페이지네이션이 적용된 API 응답 */
interface PaginatedResponse<T> extends ApiResponse<T[]> {
  /** 페이지네이션 메타 정보 */
  pagination: {
    /** 현재 페이지 번호 (1부터 시작) */
    page: number;
    /** 페이지당 항목 수 */
    size: number;
    /** 전체 항목 수 */
    totalItems: number;
    /** 전체 페이지 수 */
    totalPages: number;
  };
}

/** 정렬 방향 */
type SortOrder = 'asc' | 'desc';

/** 정렬 옵션 */
interface SortOption {
  /** 정렬 대상 필드명 */
  field: string;
  /** 정렬 방향 */
  order: SortOrder;
}

/** 필터 옵션 - 동적 쿼리 조건 */
interface FilterOption {
  /** 필터 대상 필드명 */
  field: string;
  /** 비교 연산자 */
  operator: 'eq' | 'ne' | 'gt' | 'lt' | 'gte' | 'lte' | 'in' | 'like' | 'between';
  /** 비교 값 */
  value: any;
}
```

**Name: Andrej Karpathy**
`FilterOption`의 `value`가 `any` 타입인 것이 조금 걸립니다. 연산자에 따라 값의 타입이 달라지니까 조건부 타입으로 더 엄격하게 정의할 수 있지 않을까요?

**Name: Robert C. Martin (Uncle Bob)**
좋은 지적입니다. 하지만 런타임에서 동적 필터를 처리해야 하므로 `any`를 쓰되, 각 도메인별 DTO에서 Zod 같은 런타임 검증 라이브러리로 실제 값을 검증하는 방식이 현실적입니다. 타입 시스템의 한계를 인정하고 런타임 검증으로 보완하는 전략이죠.

**Name: Martin Fowler**
동의합니다. 과도한 타입 체조(type gymnastics)는 오히려 코드 가독성을 해칩니다. 실용적인 접근이 맞습니다.

### 7.3.2 Auth 도메인 타입

**Name: Robert C. Martin (Uncle Bob)**
인증/인가 도메인은 시스템의 가장 핵심적인 부분입니다. 사용자, 역할, 권한, 그리고 위임과 승인 워크플로우까지 포함합니다.

```typescript
/** 시스템 사용자 */
interface User {
  /** 사용자 고유 식별자 (UUID) */
  id: string;
  /** 이메일 (OIDC에서 제공) */
  email: string;
  /** 사용자 이름 */
  name: string;
  /** 소속 부서 */
  department?: string;
  /** 직위 */
  position?: string;
  /** 계정 상태 */
  status: 'active' | 'inactive' | 'locked';
  /** 시스템 수준 역할 목록 */
  systemRoles: SystemRole[];
  /** 마지막 로그인 일시 (ISO 8601) */
  lastLoginAt?: string;
  /** 생성 일시 (ISO 8601) */
  createdAt: string;
  /** 수정 일시 (ISO 8601) */
  updatedAt: string;
}

/** 시스템 수준 역할 - 전역 권한 제어 */
type SystemRole = 'SYSTEM_ADMIN' | 'AUDITOR' | 'USER';

/** 역할 정의 - 시스템/프로젝트 범위 */
interface Role {
  /** 역할 고유 식별자 */
  id: string;
  /** 역할 이름 */
  name: string;
  /** 역할 설명 */
  description: string;
  /** 역할 범위: system(전역) 또는 project(프로젝트 한정) */
  scope: 'system' | 'project';
  /** 템플릿 여부 (복제 가능한 기본 역할) */
  isTemplate: boolean;
  /** 이 역할에 포함된 권한 목록 */
  permissions: Permission[];
  /** 생성 일시 */
  createdAt: string;
}

/** 단일 권한 - 리소스와 액션의 조합 */
interface Permission {
  /** 권한 고유 식별자 */
  id: string;
  /**
   * 리소스 식별자
   * 형식: "{도메인}:{리소스}" 예: "portal:dashboard", "auth:roles", "master:codes"
   */
  resource: string;
  /**
   * 액션 식별자
   * 형식: "read" | "write" | "delete" | "manage"
   */
  action: string;
  /** 권한 설명 */
  description: string;
}

/** 리소스 정의 - 권한의 대상이 되는 자원 */
interface Resource {
  /** 리소스 고유 식별자 */
  id: string;
  /** 리소스 코드 (예: "portal:dashboard") */
  code: string;
  /** 리소스 표시 이름 */
  name: string;
  /** 리소스 유형 */
  type: 'menu' | 'api' | 'data' | 'action';
  /** 상위 리소스 ID (계층 구조) */
  parentId?: string;
}

/** 프로젝트 */
interface Project {
  /** 프로젝트 고유 식별자 */
  id: string;
  /** 프로젝트 코드 (고유, 영문) */
  code: string;
  /** 프로젝트 이름 */
  name: string;
  /** 프로젝트 설명 */
  description?: string;
  /** 프로젝트 상태 */
  status: 'active' | 'archived';
  /** 생성 일시 */
  createdAt: string;
}

/** 프로젝트 멤버 - 사용자와 프로젝트의 연결 */
interface ProjectMember {
  /** 사용자 ID */
  userId: string;
  /** 프로젝트 ID */
  projectId: string;
  /** 이 프로젝트에서의 역할 목록 */
  roles: Role[];
  /** 참여 일시 */
  joinedAt: string;
}

/** JWT 토큰 페이로드 */
interface TokenPayload {
  /** Subject - 사용자 ID */
  sub: string;
  /** 사용자 이메일 */
  email: string;
  /** 사용자 이름 */
  name: string;
  /** 시스템 역할 */
  systemRoles: SystemRole[];
  /** 프로젝트별 역할 매핑 { projectId: [roleName, ...] } */
  projectRoles: Record<string, string[]>;
  /** Issued At (발행 시각, Unix timestamp) */
  iat: number;
  /** Expiration (만료 시각, Unix timestamp) */
  exp: number;
  /** JWT ID (고유 토큰 식별자) */
  jti: string;
}

/** ABAC 정책 조건 */
interface PolicyCondition {
  /** 평가 대상 속성 */
  attribute: string;
  /** 비교 연산자 */
  operator: string;
  /** 비교 값 */
  value: any;
}

/** 권한 위임 */
interface Delegation {
  /** 위임 고유 식별자 */
  id: string;
  /** 위임자 (권한을 넘기는 사람) ID */
  delegatorId: string;
  /** 수임자 (권한을 받는 사람) ID */
  delegateeId: string;
  /** 위임 대상 역할 ID */
  roleId: string;
  /** 프로젝트 범위 (없으면 시스템 수준) */
  projectId?: string;
  /** 위임 사유 */
  reason: string;
  /** 위임 시작일 (ISO 8601) */
  startDate: string;
  /** 위임 종료일 (ISO 8601) */
  endDate: string;
  /** 위임 상태 */
  status: 'active' | 'expired' | 'revoked';
}

/** 승인 요청 */
interface ApprovalRequest {
  /** 승인 요청 고유 식별자 */
  id: string;
  /** 승인 요청 유형 */
  type: 'role_assignment' | 'delegation' | 'permission_change';
  /** 요청자 ID */
  requesterId: string;
  /** 대상 사용자 ID (역할 할당 등) */
  targetUserId?: string;
  /** 요청 상세 내용 */
  details: Record<string, any>;
  /** 요청 상태 */
  status: 'pending' | 'approved' | 'rejected';
  /** 승인 단계 목록 (다단계 승인) */
  steps: ApprovalStep[];
  /** 생성 일시 */
  createdAt: string;
}

/** 승인 단계 */
interface ApprovalStep {
  /** 단계 순서 (1부터 시작) */
  order: number;
  /** 승인자 ID */
  approverId: string;
  /** 단계 상태 */
  status: 'pending' | 'approved' | 'rejected';
  /** 승인/반려 코멘트 */
  comment?: string;
  /** 결정 일시 */
  decidedAt?: string;
}
```

**Name: Andrew Ng**
`TokenPayload`에서 `projectRoles`를 `Record<string, string[]>`로 정의하셨는데, 토큰 크기가 커질 우려가 있습니다. 프로젝트가 많은 사용자의 경우 JWT 사이즈가 문제될 수 있어요.

**Name: Robert C. Martin (Uncle Bob)**
맞습니다. 실제 구현 시 토큰에는 현재 활성 프로젝트의 역할만 포함하고, 전체 프로젝트 역할은 별도 API로 조회하는 방식을 권장합니다. 타입 정의는 최대 범위를 표현하되, 실제 페이로드는 최소화해야 합니다.

**Name: Bruce Schneier**
보안 관점에서 `PolicyCondition`의 `value`도 `any` 타입인데, 이건 인젝션 공격의 벡터가 될 수 있습니다. 런타임에서 반드시 화이트리스트 기반 검증이 필요합니다.

**Name: Robert C. Martin (Uncle Bob)**
절대적으로 동의합니다. 타입 정의는 "계약"이고, 보안 검증은 "이행"입니다. 둘 다 필요합니다.

### 7.3.3 Master 도메인 타입

**Name: Robert C. Martin (Uncle Bob)**
기준정보 도메인은 가장 복잡한 타입 구조를 가집니다. 코드 그룹, 코드, 버전 관리, 변경 요청, 벌크 임포트까지 포함합니다.

```typescript
/** 코드 그룹 - 코드 분류의 최상위 단위 */
interface CodeGroup {
  /** 그룹 고유 식별자 */
  id: string;
  /** 그룹 코드 (영문, 고유) */
  code: string;
  /** 다국어 이름 */
  names: LocalizedName[];
  /** 그룹 설명 */
  description?: string;
  /** 범위: 공통 또는 프로젝트 전용 */
  scope: 'common' | 'project';
  /** 프로젝트 전용인 경우 프로젝트 ID */
  projectId?: string;
  /** 최대 계층 깊이 */
  maxDepth: number;
  /** 확장 속성 스키마 정의 */
  attributeSchema: AttributeDefinition[];
  /** 그룹 상태 */
  status: 'active' | 'inactive';
  /** 낙관적 잠금용 버전 */
  version: number;
  /** 생성 일시 */
  createdAt: string;
  /** 수정 일시 */
  updatedAt: string;
}

/** 기준정보 코드 */
interface MasterCode {
  /** 코드 고유 식별자 */
  id: string;
  /** 소속 그룹 ID */
  groupId: string;
  /** 코드 값 (그룹 내 고유) */
  code: string;
  /** 다국어 이름 */
  names: LocalizedName[];
  /** 상위 코드 ID (계층 구조) */
  parentId?: string;
  /** 계층 깊이 (0부터 시작) */
  depth: number;
  /** 정렬 순서 */
  sortOrder: number;
  /** 확장 속성 값 (그룹의 attributeSchema에 따름) */
  attributes: Record<string, any>;
  /** 코드 상태 - 라이프사이클 관리 */
  status: 'draft' | 'active' | 'deprecated' | 'archived';
  /** 유효 시작일 */
  effectiveFrom?: string;
  /** 유효 종료일 */
  effectiveTo?: string;
  /** 낙관적 잠금용 버전 */
  version: number;
  /** 생성 일시 */
  createdAt: string;
  /** 수정 일시 */
  updatedAt: string;
}

/** 다국어 이름 */
interface LocalizedName {
  /** 로케일 코드 (예: "ko", "en", "ja") */
  locale: string;
  /** 전체 이름 */
  name: string;
  /** 축약 이름 */
  shortName?: string;
  /** 설명 */
  description?: string;
}

/** 코드 버전 이력 */
interface CodeVersion {
  /** 버전 이력 고유 식별자 */
  id: string;
  /** 대상 코드 ID */
  codeId: string;
  /** 버전 번호 */
  version: number;
  /** 변경 상세 내역 */
  changes: ChangeDetail[];
  /** 변경자 ID */
  changedBy: string;
  /** 변경 일시 */
  changedAt: string;
}

/** 변경 상세 - 필드 단위 변경 추적 */
interface ChangeDetail {
  /** 변경된 필드명 */
  field: string;
  /** 변경 전 값 */
  oldValue: any;
  /** 변경 후 값 */
  newValue: any;
}

/** 확장 속성 정의 - 그룹별 커스텀 필드 스키마 */
interface AttributeDefinition {
  /** 속성 키 (영문) */
  key: string;
  /** 다국어 라벨 */
  label: LocalizedName[];
  /** 속성 데이터 타입 */
  type: 'string' | 'number' | 'boolean' | 'date' | 'select';
  /** 필수 여부 */
  required: boolean;
  /** select 타입인 경우 선택 가능 옵션 */
  options?: string[];
}

/** 변경 요청 - 승인 워크플로우 연동 */
interface ChangeRequest {
  /** 변경 요청 고유 식별자 */
  id: string;
  /** 변경 유형 */
  type: 'create' | 'update' | 'deprecate' | 'delete';
  /** 대상 유형 */
  targetType: 'code' | 'group';
  /** 대상 ID (신규 생성 시 없음) */
  targetId?: string;
  /** 변경 내용 */
  payload: Record<string, any>;
  /** 요청자 ID */
  requesterId: string;
  /** 요청 상태 */
  status: 'draft' | 'pending' | 'approved' | 'rejected' | 'applied';
  /** 영향도 분석 결과 */
  impactAnalysis?: ImpactAnalysis;
  /** 생성 일시 */
  createdAt: string;
}

/** 영향도 분석 결과 */
interface ImpactAnalysis {
  /** 영향받는 프로젝트 ID 목록 */
  affectedProjects: string[];
  /** 영향받는 코드 수 */
  affectedCodes: number;
  /** 위험 수준 */
  riskLevel: 'low' | 'medium' | 'high';
}

/** 벌크 임포트 작업 */
interface BulkImportJob {
  /** 작업 고유 식별자 */
  id: string;
  /** 대상 그룹 ID */
  groupId: string;
  /** 업로드된 파일명 */
  fileName: string;
  /** 전체 행 수 */
  totalRows: number;
  /** 처리된 행 수 */
  processedRows: number;
  /** 성공 행 수 */
  successRows: number;
  /** 오류 행 수 */
  errorRows: number;
  /** 작업 상태 */
  status: 'uploaded' | 'validating' | 'queued' | 'processing' | 'completed' | 'failed';
  /** 오류 목록 */
  errors?: BulkImportError[];
  /** 생성자 ID */
  createdBy: string;
  /** 생성 일시 */
  createdAt: string;
  /** 완료 일시 */
  completedAt?: string;
}

/** 벌크 임포트 오류 상세 */
interface BulkImportError {
  /** 오류 발생 행 번호 */
  row: number;
  /** 오류 필드명 */
  field: string;
  /** 오류 메시지 */
  message: string;
  /** 입력된 값 */
  value?: any;
}
```

**Name: Martin Kleppmann**
`CodeGroup`과 `MasterCode` 모두 `version` 필드가 있는데, 이건 낙관적 동시성 제어(Optimistic Concurrency Control)를 위한 거죠? 이 패턴이면 동시 편집 충돌을 깔끔하게 감지할 수 있습니다. PUT 요청 시 현재 버전을 함께 보내고, 서버에서 버전 불일치 시 409 Conflict를 반환하면 됩니다.

**Name: Robert C. Martin (Uncle Bob)**
정확합니다. `version` 필드는 ETag 헤더와 함께 사용됩니다. 분산 환경에서 데이터 정합성을 보장하는 핵심 메커니즘입니다.

**Name: Jakob Nielsen**
`LocalizedName`에 `shortName`이 optional인 것이 좋습니다. UI에서 테이블 컬럼이나 태그처럼 좁은 공간에서는 축약 이름이 필수인데, 없으면 `name`을 잘라서 쓰면 됩니다. 프론트엔드에서 `shortName ?? name` 패턴으로 처리하면 깔끔합니다.

### 7.3.4 Portal 도메인 타입

**Name: Robert C. Martin (Uncle Bob)**
포탈 도메인은 마이크로 프론트엔드 쉘의 상태와 앱 매니페스트를 정의합니다.

```typescript
/** 마이크로 프론트엔드 앱 매니페스트 */
interface AppManifest {
  /** 앱 고유 식별자 */
  id: string;
  /** 앱 이름 */
  name: string;
  /** 앱 기본 경로 (예: "/master", "/auth") */
  basePath: string;
  /** 앱 진입점 URL */
  entry: string;
  /** 라우트 설정 목록 */
  routes: RouteConfig[];
  /** 앱 접근에 필요한 권한 */
  requiredPermissions: string[];
  /** 앱 버전 (SemVer) */
  version: string;
  /** 프레임워크 유형 */
  framework: 'react' | 'vue' | 'angular' | 'svelte' | 'other';
  /** 로딩 전략 */
  loadStrategy: 'module-federation' | 'single-spa' | 'iframe';
  /** 헬스체크 엔드포인트 */
  healthCheck?: string;
  /** 추가 메타데이터 */
  metadata?: Record<string, any>;
}

/** 라우트 설정 */
interface RouteConfig {
  /** 라우트 경로 */
  path: string;
  /** 라우트 이름 (메뉴 표시용) */
  name: string;
  /** 아이콘 식별자 */
  icon?: string;
  /** 메뉴 그룹 */
  group?: string;
  /** 정렬 순서 */
  order?: number;
  /** 사이드바 메뉴에 표시 여부 */
  showInSidebar: boolean;
  /** 이 라우트 접근에 필요한 권한 */
  requiredPermissions?: string[];
}

/** 쉘 전역 상태 */
interface ShellState {
  /** 인증 상태 */
  auth: {
    /** 현재 로그인 사용자 */
    user: User | null;
    /** 액세스 토큰 */
    token: string | null;
    /** 인증 여부 */
    isAuthenticated: boolean;
  };
  /** 네비게이션 상태 */
  navigation: {
    /** 현재 활성 앱 ID */
    currentApp: string | null;
    /** 현재 경로 */
    currentPath: string;
    /** 브레드크럼 목록 */
    breadcrumbs: Breadcrumb[];
  };
  /** UI 상태 */
  ui: {
    /** 사이드바 접힘 여부 */
    sidebarCollapsed: boolean;
    /** 테마 */
    theme: 'light' | 'dark';
    /** 현재 로케일 */
    locale: string;
  };
  /** 알림 목록 */
  notifications: Notification[];
}

/** 알림 */
interface Notification {
  /** 알림 고유 식별자 */
  id: string;
  /** 알림 유형 */
  type: 'info' | 'warning' | 'error' | 'success' | 'approval';
  /** 알림 제목 */
  title: string;
  /** 알림 내용 */
  message: string;
  /** 연결 링크 (클릭 시 이동) */
  link?: string;
  /** 읽음 여부 */
  read: boolean;
  /** 생성 일시 */
  createdAt: string;
}

/** 브레드크럼 항목 */
interface Breadcrumb {
  /** 표시 텍스트 */
  label: string;
  /** 이동 경로 */
  path: string;
}
```

**Name: Martin Fowler**
`AppManifest`의 `loadStrategy`에서 세 가지 옵션을 제공하는 것이 좋습니다. Module Federation이 주 전략이지만, 레거시 앱을 iframe으로 감싸거나 single-spa로 통합하는 유연성을 확보할 수 있습니다. 이것이 바로 점진적 마이그레이션을 가능하게 하는 설계입니다.

**Name: Andrej Karpathy**
`ShellState`가 전역 스토어의 인터페이스 역할을 하는 거군요. 마이크로 프론트엔드 간 상태 공유가 이 인터페이스를 통해 이루어지면 앱 간 결합도를 최소화할 수 있습니다.

### 7.3.5 Event 타입

**Name: Robert C. Martin (Uncle Bob)**
마지막으로 도메인 이벤트 타입입니다. 각 도메인에서 발행하는 이벤트를 타입으로 정의하여 이벤트 기반 통합의 계약을 명확히 합니다.

```typescript
/** 도메인 이벤트 기본 타입 */
interface DomainEvent<T = any> {
  /** 이벤트 고유 식별자 (UUID) */
  id: string;
  /** 이벤트 유형 (예: "auth.role_changed") */
  type: string;
  /** 이벤트 발행 소스 (예: "auth-service") */
  source: string;
  /** 이벤트 페이로드 */
  payload: T;
  /** 이벤트 발생 일시 (ISO 8601) */
  occurredAt: string;
  /** 상관관계 ID (요청 추적용) */
  correlationId?: string;
}

/** 인증 도메인 이벤트 */
type AuthEvent = DomainEvent<{
  /** 인증 이벤트 액션 */
  action: 'login' | 'logout' | 'role_changed' | 'permission_changed';
  /** 대상 사용자 ID */
  userId: string;
  /** 추가 상세 정보 */
  details: Record<string, any>;
}>;

/** 기준정보 도메인 이벤트 */
type MasterEvent = DomainEvent<{
  /** 기준정보 이벤트 액션 */
  action: 'code_created' | 'code_updated' | 'code_deprecated' | 'group_updated';
  /** 대상 그룹 코드 */
  groupCode: string;
  /** 대상 코드 ID */
  codeId?: string;
  /** 변경 상세 */
  changes?: ChangeDetail[];
}>;

/** 포탈 도메인 이벤트 */
type PortalEvent = DomainEvent<{
  /** 포탈 이벤트 액션 */
  action: 'app_registered' | 'app_updated' | 'app_removed';
  /** 대상 앱 ID */
  appId: string;
}>;
```

**Name: Martin Kleppmann**
이벤트 타입 설계가 훌륭합니다. `correlationId`가 있어서 분산 추적이 가능하고, `DomainEvent` 제네릭으로 페이로드를 타입 안전하게 다룰 수 있습니다. 한 가지 제안하자면, 이벤트 스키마 버전 필드(`schemaVersion`)도 추가하면 이벤트 구조가 진화할 때 호환성을 관리하기 쉽습니다.

**Name: Robert C. Martin (Uncle Bob)**
좋은 제안이지만 현재 MVP 단계에서는 이벤트 구조가 안정화될 때까지 버전 관리 없이 시작하고, 필요해지는 시점에 추가하는 것이 YAGNI 원칙에 맞습니다.

**Name: Kent Beck (사회자)**
타입 안전성과 실용성 사이의 균형을 잘 잡으셨습니다. enum 대신 union type을 선택한 이유를 정리해 주시겠습니까?

**Name: Robert C. Martin (Uncle Bob)**
TypeScript에서 `enum`은 런타임에 객체를 생성하고 트리 쉐이킹이 어렵습니다. 반면 union type(`'active' | 'inactive'`)은 컴파일 시점에만 존재하고 번들 사이즈에 영향을 주지 않습니다. 또한 union type은 패턴 매칭에서 더 자연스럽게 동작합니다. 이것이 현대 TypeScript의 권장 패턴입니다.

**Name: Andrej Karpathy**
Optional 필드와 Required 필드의 구분 기준도 명확히 해두면 좋겠습니다. 현재 설계에서 `?` 마크가 붙은 필드들은 대부분 "생성 시 선택, 조회 시 null 가능"의 의미를 가지는데, 이 규칙이 일관되게 적용되어 있어서 좋습니다.

> **합의:** 모든 도메인의 TypeScript 인터페이스를 JSDoc 주석과 함께 정의 완료. enum 대신 union type 사용, Optional 필드는 `?`로 명시, 낙관적 잠금을 위한 `version` 필드 포함, 이벤트 타입은 제네릭 기반 `DomainEvent<T>` 패턴 채택. 런타임 검증은 Zod 스키마로 보완하며, 과도한 타입 체조보다 실용성을 우선시하기로 합의.

---

## 토의 4: API 엔드포인트 + 입출력 스키마 (7.4)

**Name: Kent Beck (사회자)**
네 번째 주제입니다. 앞서 정의한 타입들을 기반으로 실제 REST API 엔드포인트를 설계해야 합니다. Fowler와 Uncle Bob이 함께 리드해 주시죠.

**Name: Martin Fowler**
RESTful API 설계에서 가장 중요한 것은 일관성입니다. URL 구조, HTTP 메서드 사용, 응답 형식이 모든 도메인에서 동일한 규칙을 따라야 합니다. 버전 관리는 URL 경로 방식(`/api/v1/`)을 채택합니다.

**Name: Robert C. Martin (Uncle Bob)**
동의합니다. 각 엔드포인트가 하나의 책임만 가지도록 설계하겠습니다. 입출력 스키마도 앞서 정의한 인터페이스를 기반으로 명확히 합니다.

### 7.4.1 Auth API (`/api/v1/auth`)

**Name: Martin Fowler**
인증 API는 OIDC 플로우를 지원하며, 최소한의 엔드포인트로 구성합니다.

| Method | Path | Description | Request Body / Query | Response | Auth 필요 |
|--------|------|-------------|---------------------|----------|-----------|
| POST | `/login` | OIDC 로그인 시작 | `{ returnUrl: string }` | `ApiResponse<{ redirectUrl: string }>` | No |
| POST | `/callback` | OIDC 콜백 처리 | `{ code: string, state: string }` | `ApiResponse<AuthTokens>` | No |
| POST | `/refresh` | 토큰 갱신 | `{ refreshToken: string }` | `ApiResponse<AuthTokens>` | No |
| POST | `/logout` | 로그아웃 | - | `ApiResponse<void>` | Yes |
| GET | `/me` | 내 정보 조회 | - | `ApiResponse<User>` | Yes |

```typescript
/** 인증 토큰 응답 */
interface AuthTokens {
  /** JWT 액세스 토큰 */
  accessToken: string;
  /** 리프레시 토큰 */
  refreshToken: string;
  /** 액세스 토큰 만료 시간 (초) */
  expiresIn: number;
  /** 토큰 타입 (항상 "Bearer") */
  tokenType: 'Bearer';
}
```

**Name: Bruce Schneier**
`/login`, `/callback`, `/refresh`는 인증 없이 접근 가능하지만, Rate Limiting은 반드시 적용해야 합니다. 특히 `/login`은 무차별 대입 공격의 대상이 되기 쉬우니 IP 기반 제한도 함께 적용해야 합니다.

**Name: Martin Fowler**
맞습니다. 모든 인증 엔드포인트에는 분당 10회 제한을 두겠습니다.

### 7.4.2 Permission API (`/api/v1/permissions`)

**Name: Robert C. Martin (Uncle Bob)**
권한 관리 API는 실시간 권한 체크와 역할/위임/승인 관리를 포함합니다.

| Method | Path | Description | Request Body / Query | Response | Auth 필요 |
|--------|------|-------------|---------------------|----------|-----------|
| POST | `/check` | 단일 권한 체크 | `{ resource: string, action: string, projectId?: string }` | `ApiResponse<{ allowed: boolean, reason?: string }>` | Yes |
| POST | `/batch-check` | 배치 권한 체크 | `{ checks: Array<{ resource: string, action: string, projectId?: string }> }` | `ApiResponse<CheckResult[]>` | Yes |
| GET | `/roles` | 역할 목록 조회 | `?scope=system\|project&page=1&size=20` | `PaginatedResponse<Role>` | Yes |
| POST | `/roles` | 역할 생성 | `CreateRoleDto` | `ApiResponse<Role>` | Yes (Admin) |
| PUT | `/roles/:id` | 역할 수정 | `UpdateRoleDto` | `ApiResponse<Role>` | Yes (Admin) |
| DELETE | `/roles/:id` | 역할 삭제 | - | `ApiResponse<void>` | Yes (Admin) |
| POST | `/roles/:id/clone` | 역할 복제 | `{ name: string, projectId: string }` | `ApiResponse<Role>` | Yes (Admin) |
| GET | `/delegations` | 위임 목록 조회 | `?status=active\|expired\|revoked&page=1&size=20` | `PaginatedResponse<Delegation>` | Yes |
| POST | `/delegations` | 위임 생성 | `CreateDelegationDto` | `ApiResponse<Delegation>` | Yes |
| POST | `/approvals` | 승인 요청 생성 | `CreateApprovalDto` | `ApiResponse<ApprovalRequest>` | Yes |
| PUT | `/approvals/:id/decide` | 승인/반려 처리 | `{ decision: 'approved' \| 'rejected', comment?: string }` | `ApiResponse<ApprovalRequest>` | Yes |

```typescript
/** 권한 체크 결과 */
interface CheckResult {
  /** 체크 대상 리소스 */
  resource: string;
  /** 체크 대상 액션 */
  action: string;
  /** 허용 여부 */
  allowed: boolean;
  /** 사유 (거부 시) */
  reason?: string;
}

/** 역할 생성 DTO */
interface CreateRoleDto {
  name: string;
  description: string;
  scope: 'system' | 'project';
  permissionIds: string[];
}

/** 역할 수정 DTO */
interface UpdateRoleDto {
  name?: string;
  description?: string;
  permissionIds?: string[];
}

/** 위임 생성 DTO */
interface CreateDelegationDto {
  delegateeId: string;
  roleId: string;
  projectId?: string;
  reason: string;
  startDate: string;
  endDate: string;
}

/** 승인 요청 생성 DTO */
interface CreateApprovalDto {
  type: 'role_assignment' | 'delegation' | 'permission_change';
  targetUserId?: string;
  details: Record<string, any>;
}
```

**Name: Andrew Ng**
`/batch-check`는 프론트엔드에서 페이지 로딩 시 여러 권한을 한 번에 체크하는 데 매우 유용합니다. 네트워크 왕복을 최소화하는 좋은 설계입니다.

**Name: Martin Fowler**
맞습니다. 권한 체크는 매 페이지 전환마다 발생하므로 성능이 중요합니다. 서버 측에서도 Redis 캐시를 활용하여 응답 시간을 최소화해야 합니다.

### 7.4.3 Master API (`/api/v1/master`)

**Name: Martin Fowler**
기준정보 API는 가장 많은 엔드포인트를 가집니다. CRUD, 상태 관리, 벌크 작업, 검색까지 포함합니다.

| Method | Path | Description | Request Body / Query | Response | Auth 필요 |
|--------|------|-------------|---------------------|----------|-----------|
| POST | `/groups` | 코드 그룹 생성 | `CreateGroupDto` | `ApiResponse<CodeGroup>` | Yes (Admin) |
| GET | `/groups` | 코드 그룹 목록 | `?scope=common\|project&status=active\|inactive&page=1&size=20` | `PaginatedResponse<CodeGroup>` | Yes |
| GET | `/groups/:id` | 코드 그룹 상세 | - | `ApiResponse<CodeGroup>` | Yes |
| GET | `/codes` | 코드 목록 조회 | `?groupId=&status=&parent=&page=1&size=20` | `PaginatedResponse<MasterCode>` | Yes |
| POST | `/codes` | 코드 생성 | `CreateCodeDto` | `ApiResponse<MasterCode>` | Yes |
| PUT | `/codes/:id` | 코드 수정 | `UpdateCodeDto` | `ApiResponse<MasterCode>` | Yes |
| PATCH | `/codes/:id/status` | 코드 상태 변경 | `{ status: string, reason: string }` | `ApiResponse<MasterCode>` | Yes |
| GET | `/lookup/:groupCode` | 코드 조회 (캐시) | `?locale=ko&active=true` | `ApiResponse<MasterCode[]>` (ETag 포함) | Yes |
| POST | `/bulk/import` | 벌크 업로드 | `multipart/form-data` (file + groupId) | `ApiResponse<BulkImportJob>` | Yes (Admin) |
| GET | `/bulk/jobs/:id` | 벌크 작업 상태 | - | `ApiResponse<BulkImportJob>` | Yes |
| GET | `/bulk/export/:groupId` | 엑셀 다운로드 | `?format=xlsx\|csv` | File download (`Content-Disposition`) | Yes |
| GET | `/search` | 통합 검색 | `?q=검색어&type=code\|group&page=1&size=20` | `PaginatedResponse<SearchResult>` | Yes |

```typescript
/** 코드 그룹 생성 DTO */
interface CreateGroupDto {
  code: string;
  names: LocalizedName[];
  description?: string;
  scope: 'common' | 'project';
  projectId?: string;
  maxDepth: number;
  attributeSchema: AttributeDefinition[];
}

/** 코드 생성 DTO */
interface CreateCodeDto {
  groupId: string;
  code: string;
  names: LocalizedName[];
  parentId?: string;
  sortOrder: number;
  attributes?: Record<string, any>;
  effectiveFrom?: string;
  effectiveTo?: string;
}

/** 코드 수정 DTO */
interface UpdateCodeDto {
  names?: LocalizedName[];
  sortOrder?: number;
  attributes?: Record<string, any>;
  effectiveFrom?: string;
  effectiveTo?: string;
  /** 낙관적 잠금 - 현재 버전 필수 */
  version: number;
}

/** 검색 결과 */
interface SearchResult {
  /** 결과 유형 */
  type: 'code' | 'group';
  /** 대상 ID */
  id: string;
  /** 표시 이름 */
  name: string;
  /** 그룹 코드 */
  groupCode: string;
  /** 코드 값 */
  code: string;
  /** 매칭된 텍스트 하이라이트 */
  highlight?: string;
}
```

**Name: Martin Kleppmann**
`/lookup/:groupCode` 엔드포인트에 ETag를 사용한 것이 핵심입니다. 프론트엔드에서 `If-None-Match` 헤더로 캐시 유효성을 확인하면 304 Not Modified 응답으로 네트워크 대역폭을 절약할 수 있습니다. 기준정보 특성상 자주 변경되지 않으므로 캐시 효율이 매우 높을 것입니다.

**Name: Robert C. Martin (Uncle Bob)**
PATCH를 상태 변경에만 사용하고 PUT을 전체 수정에 사용하는 패턴도 주목해 주세요. HTTP 메서드의 의미론적 사용이 API의 명확성을 높입니다.

**Name: Jakob Nielsen**
`/search` 엔드포인트에서 `highlight` 필드를 제공하는 것이 UX 관점에서 매우 중요합니다. 사용자가 검색 결과에서 왜 이 항목이 매칭되었는지 즉시 파악할 수 있게 해줍니다.

### 7.4.4 Portal API (`/api/v1/portal`)

**Name: Martin Fowler**
포탈 API는 쉘 앱에서 사용하는 관리/운영 엔드포인트입니다.

| Method | Path | Description | Request Body / Query | Response | Auth 필요 |
|--------|------|-------------|---------------------|----------|-----------|
| GET | `/apps` | 등록된 앱 목록 | - | `ApiResponse<AppManifest[]>` | Yes |
| GET | `/apps/:id` | 앱 상세 조회 | - | `ApiResponse<AppManifest>` | Yes |
| POST | `/apps` | 앱 등록 | `AppManifest` | `ApiResponse<AppManifest>` | Yes (Admin) |
| GET | `/dashboard` | 대시보드 집약 데이터 | - | `ApiResponse<DashboardData>` | Yes |
| GET | `/search` | 통합 검색 | `?q=검색어&scope=all\|codes\|users\|apps` | `ApiResponse<SearchResult[]>` | Yes |
| GET | `/notifications` | 알림 목록 | `?read=true\|false&page=1&size=20` | `PaginatedResponse<Notification>` | Yes |
| PATCH | `/notifications/:id/read` | 알림 읽음 처리 | - | `ApiResponse<void>` | Yes |

```typescript
/** 대시보드 집약 데이터 */
interface DashboardData {
  /** 통계 요약 */
  stats: {
    /** 전체 코드 그룹 수 */
    totalGroups: number;
    /** 활성 코드 수 */
    activeCodes: number;
    /** 대기중 승인 요청 수 */
    pendingApprovals: number;
    /** 최근 변경 수 (24시간) */
    recentChanges: number;
  };
  /** 최근 활동 */
  recentActivities: Activity[];
  /** 내 대기 작업 */
  myPendingTasks: PendingTask[];
}

/** 활동 이력 */
interface Activity {
  id: string;
  type: string;
  description: string;
  userId: string;
  userName: string;
  occurredAt: string;
}

/** 대기 작업 */
interface PendingTask {
  id: string;
  type: 'approval' | 'review' | 'import';
  title: string;
  link: string;
  createdAt: string;
}
```

### 7.4.5 표준 에러 응답 및 에러 코드 체계

**Name: Robert C. Martin (Uncle Bob)**
모든 API는 동일한 에러 응답 형식을 따릅니다. 에러 코드는 도메인별로 체계적으로 분류합니다.

```typescript
/** 표준 에러 응답 예시 */
// HTTP 400 Bad Request
{
  "status": "error",
  "data": null,
  "error": {
    "code": "VALIDATION_001",
    "message": "입력값 검증에 실패했습니다.",
    "details": {
      "fields": {
        "email": "유효한 이메일 형식이 아닙니다.",
        "name": "필수 입력 항목입니다."
      }
    }
  },
  "meta": {
    "requestId": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2025-01-15T09:30:00Z"
  }
}
```

**에러 코드 테이블:**

| 코드 | HTTP 상태 | 설명 |
|------|----------|------|
| `AUTH_001` | 401 | 인증 토큰이 없거나 만료됨 |
| `AUTH_002` | 401 | 유효하지 않은 리프레시 토큰 |
| `AUTH_003` | 403 | 접근 권한 없음 |
| `AUTH_004` | 403 | 계정이 잠김 (locked) |
| `AUTH_005` | 409 | 위임 기간 충돌 |
| `VALIDATION_001` | 400 | 입력값 검증 실패 |
| `VALIDATION_002` | 400 | 필수 필드 누락 |
| `VALIDATION_003` | 400 | 데이터 형식 불일치 |
| `RESOURCE_001` | 404 | 리소스를 찾을 수 없음 |
| `RESOURCE_002` | 409 | 리소스 충돌 (중복 코드 등) |
| `RESOURCE_003` | 409 | 버전 충돌 (낙관적 잠금 실패) |
| `MASTER_001` | 400 | 코드 그룹 제약 조건 위반 |
| `MASTER_002` | 400 | 계층 깊이 초과 |
| `MASTER_003` | 422 | 상태 전이 규칙 위반 |
| `MASTER_004` | 400 | 벌크 임포트 파일 형식 오류 |
| `MASTER_005` | 422 | 영향도 분석 필요 (고위험 변경) |
| `PORTAL_001` | 400 | 앱 매니페스트 형식 오류 |
| `PORTAL_002` | 409 | 앱 basePath 충돌 |
| `SYSTEM_001` | 500 | 내부 서버 오류 |
| `SYSTEM_002` | 503 | 서비스 일시 불가 |
| `SYSTEM_003` | 429 | 요청 한도 초과 (Rate Limit) |

**Name: Bruce Schneier**
에러 메시지에서 보안에 민감한 내부 정보가 노출되지 않도록 주의해야 합니다. 예를 들어 `AUTH_001`의 상세 메시지에 "토큰 서명 키가 일치하지 않음" 같은 내부 구현 상세를 포함하면 안 됩니다. 사용자에게는 일반적인 메시지만 제공하고, 상세 로그는 서버 측에만 기록해야 합니다.

**Name: Martin Fowler**
절대적으로 맞습니다. `details` 필드는 검증 오류 같은 클라이언트가 활용할 수 있는 정보만 포함하고, 스택 트레이스나 내부 구현 정보는 절대 포함하지 않습니다.

### 7.4.6 Rate Limiting 정책

**Name: Bruce Schneier**
API 남용 방지를 위한 Rate Limiting 정책을 명확히 정의해야 합니다.

| 카테고리 | 제한 | 적용 대상 | 헤더 |
|---------|------|----------|------|
| Auth (인증) | 10회/분 | `/api/v1/auth/*` | `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` |
| Read (읽기) | 100회/분 | GET 요청 전체 | 동일 |
| Write (쓰기) | 30회/분 | POST, PUT, PATCH, DELETE | 동일 |
| Bulk (벌크) | 5회/분 | `/api/v1/master/bulk/*` | 동일 |

```typescript
/** Rate Limit 초과 시 응답 헤더 예시 */
// HTTP 429 Too Many Requests
// X-RateLimit-Limit: 100
// X-RateLimit-Remaining: 0
// X-RateLimit-Reset: 1705312260
// Retry-After: 45
```

**Name: Andrew Ng**
Rate Limiting은 사용자 단위(토큰 기반)와 IP 단위 두 가지를 병행 적용하는 것이 좋습니다. 인증된 사용자는 토큰 기반으로, 미인증 엔드포인트(`/login`, `/callback`)는 IP 기반으로 제한해야 합니다.

**Name: Martin Fowler**
동의합니다. 또한 Rate Limit 정보를 응답 헤더로 항상 반환하여 클라이언트가 자체적으로 요청 빈도를 조절할 수 있게 해야 합니다.

### 7.4.7 API 버전 관리 및 보안 헤더

**Name: Martin Fowler**
API 버전 관리는 URL 경로 방식(`/api/v1/`)을 사용합니다. 헤더 기반 버전 관리보다 명시적이고, API 게이트웨이에서 라우팅하기도 쉽습니다.

```
# 버전 관리 규칙
- 현재 버전: /api/v1/
- 하위 호환성 파괴 시 새 버전: /api/v2/
- 이전 버전은 최소 6개월 병행 운영 후 폐기
- 폐기 예정 API는 응답 헤더에 Deprecation 경고 포함
  Deprecation: true
  Sunset: Sat, 01 Jan 2026 00:00:00 GMT
  Link: </api/v2/resource>; rel="successor-version"
```

**Name: Bruce Schneier**
모든 API 응답에 포함해야 할 보안 헤더를 정리하겠습니다.

```
# 필수 보안 헤더
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0
Content-Security-Policy: default-src 'self'; script-src 'self'
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Cache-Control: no-store (인증 관련 응답)
```

**Name: Bruce Schneier**
CORS 설정도 매우 중요합니다. 허용할 오리진을 명시적으로 화이트리스트로 관리해야 하며, 절대로 와일드카드(`*`)를 사용하면 안 됩니다.

```
# CORS 설정
Access-Control-Allow-Origin: https://portal.example.com (화이트리스트)
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-Request-ID
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
Access-Control-Expose-Headers: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, ETag
```

**Name: Martin Fowler**
CORS에서 `Access-Control-Expose-Headers`에 Rate Limit 관련 헤더와 ETag를 포함한 것이 중요합니다. 이렇게 해야 프론트엔드 JavaScript에서 이 헤더 값들을 읽을 수 있습니다.

**Name: Jakob Nielsen**
프론트엔드 관점에서 한 가지 더 추가하겠습니다. 모든 API 요청에 `X-Request-ID` 헤더를 포함하고, 응답의 `meta.requestId`로 반환하면 사용자가 문제를 보고할 때 이 ID를 전달할 수 있어 디버깅이 훨씬 수월해집니다.

**Name: Kent Beck (사회자)**
훌륭한 논의였습니다. API 설계가 일관되고 보안이 철저하며, 프론트엔드 개발자가 사용하기 편리한 구조로 완성되었습니다.

> **합의:** 4개 도메인(Auth, Permission, Master, Portal)의 전체 REST API 엔드포인트를 테이블 형식으로 정의 완료. URL 경로 기반 버전 관리(`/api/v1/`), 도메인별 에러 코드 체계(AUTH_xxx, MASTER_xxx, SYSTEM_xxx 등 19개), 4단계 Rate Limiting 정책(Auth 10/min, Read 100/min, Write 30/min, Bulk 5/min), 필수 보안 헤더 및 CORS 화이트리스트 방식을 합의. ETag 기반 캐시, 낙관적 잠금(version 필드), 표준 에러 응답 형식을 전 API에 일관 적용하기로 결정.

---

---

## 토의 5: DB 스키마 (SQL DDL) — 전체 테이블 정의 (7.5)

**Kent Beck:** 다섯 번째 주제입니다. 지금까지 아키텍처, API, 디렉토리 구조까지 정했으니, 이제 실제 데이터가 저장될 DB 스키마를 확정해야 합니다. Martin Kleppmann, Auth DB부터 시작하시죠.

**Martin Kleppmann:** 네, 두 개의 논리적 DB — Auth DB와 Master DB — 를 PostgreSQL 16 기반으로 설계합니다. 먼저 Auth DB의 핵심 테이블부터 전체 DDL을 보여드리겠습니다.

```sql
-- ============================================================
-- AUTH DATABASE DDL
-- PostgreSQL 16
-- ============================================================

-- 0. Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================================
-- 1. users
-- ============================================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    employee_id     VARCHAR(20)  NOT NULL UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    department      VARCHAR(100),
    position        VARCHAR(50),
    phone           VARCHAR(20),
    avatar_url      VARCHAR(500),
    idp_sub         VARCHAR(255) UNIQUE,          -- Keycloak subject ID
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE'
                    CHECK (status IN ('ACTIVE', 'INACTIVE', 'LOCKED', 'PENDING')),
    last_login_at   TIMESTAMPTZ,
    password_hash   VARCHAR(255),                 -- 로컬 인증 fallback용
    mfa_enabled     BOOLEAN      NOT NULL DEFAULT FALSE,
    mfa_secret      VARCHAR(255),
    failed_login_count INT       NOT NULL DEFAULT 0,
    locked_until    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_employee_id ON users (employee_id);
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_idp_sub ON users (idp_sub);
CREATE INDEX idx_users_status ON users (status);
CREATE INDEX idx_users_department ON users (department);

-- ============================================================
-- 2. roles
-- ============================================================
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code            VARCHAR(50)  NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    scope           VARCHAR(20)  NOT NULL DEFAULT 'PROJECT'
                    CHECK (scope IN ('SYSTEM', 'PROJECT')),
    is_default      BOOLEAN      NOT NULL DEFAULT FALSE,
    priority        INT          NOT NULL DEFAULT 0,
    max_members     INT,                          -- 역할당 최대 인원 제한
    created_by      UUID         REFERENCES users(id),
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_roles_code ON roles (code);
CREATE INDEX idx_roles_scope ON roles (scope);

-- ============================================================
-- 3. resources
-- ============================================================
CREATE TABLE resources (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code            VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    type            VARCHAR(30)  NOT NULL
                    CHECK (type IN ('API', 'UI', 'DATA', 'REPORT', 'FUNCTION')),
    service         VARCHAR(50)  NOT NULL,         -- 소속 서비스 (auth, mdm 등)
    description     TEXT,
    parent_id       UUID         REFERENCES resources(id),
    is_active       BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_resources_code ON resources (code);
CREATE INDEX idx_resources_type ON resources (type);
CREATE INDEX idx_resources_service ON resources (service);
CREATE INDEX idx_resources_parent ON resources (parent_id);

-- ============================================================
-- 4. permissions
-- ============================================================
CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    resource_id     UUID         NOT NULL REFERENCES resources(id) ON DELETE CASCADE,
    action          VARCHAR(20)  NOT NULL
                    CHECK (action IN ('CREATE', 'READ', 'UPDATE', 'DELETE', 'EXECUTE', 'APPROVE', 'EXPORT')),
    conditions      JSONB,                         -- 조건부 권한 (e.g., {"department": "same"})
    description     TEXT,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (resource_id, action)
);

CREATE INDEX idx_permissions_resource ON permissions (resource_id);
CREATE INDEX idx_permissions_action ON permissions (action);

-- ============================================================
-- 5. role_permissions
-- ============================================================
CREATE TABLE role_permissions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    role_id         UUID         NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID         NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    granted_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    granted_by      UUID         REFERENCES users(id),
    UNIQUE (role_id, permission_id)
);

CREATE INDEX idx_role_permissions_role ON role_permissions (role_id);
CREATE INDEX idx_role_permissions_permission ON role_permissions (permission_id);

-- ============================================================
-- 6. user_system_roles (시스템 레벨 역할)
-- ============================================================
CREATE TABLE user_system_roles (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID         NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    assigned_by     UUID         REFERENCES users(id),
    expires_at      TIMESTAMPTZ,                   -- 임시 권한 만료
    UNIQUE (user_id, role_id)
);

CREATE INDEX idx_user_system_roles_user ON user_system_roles (user_id);
CREATE INDEX idx_user_system_roles_role ON user_system_roles (role_id);
CREATE INDEX idx_user_system_roles_expires ON user_system_roles (expires_at)
    WHERE expires_at IS NOT NULL;

-- ============================================================
-- 7. projects
-- ============================================================
CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code            VARCHAR(50)  NOT NULL UNIQUE,
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE'
                    CHECK (status IN ('ACTIVE', 'ARCHIVED', 'SUSPENDED')),
    owner_id        UUID         NOT NULL REFERENCES users(id),
    settings        JSONB        NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_projects_code ON projects (code);
CREATE INDEX idx_projects_status ON projects (status);
CREATE INDEX idx_projects_owner ON projects (owner_id);

-- ============================================================
-- 8. project_members
-- ============================================================
CREATE TABLE project_members (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_id      UUID         NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    joined_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE'
                    CHECK (status IN ('ACTIVE', 'INACTIVE', 'INVITED')),
    UNIQUE (project_id, user_id)
);

CREATE INDEX idx_project_members_project ON project_members (project_id);
CREATE INDEX idx_project_members_user ON project_members (user_id);

-- ============================================================
-- 9. project_member_roles
-- ============================================================
CREATE TABLE project_member_roles (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    project_member_id UUID       NOT NULL REFERENCES project_members(id) ON DELETE CASCADE,
    role_id         UUID         NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_at     TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    assigned_by     UUID         REFERENCES users(id),
    expires_at      TIMESTAMPTZ,
    UNIQUE (project_member_id, role_id)
);

CREATE INDEX idx_pmr_member ON project_member_roles (project_member_id);
CREATE INDEX idx_pmr_role ON project_member_roles (role_id);

-- ============================================================
-- 10. permission_delegations (권한 위임)
-- ============================================================
CREATE TABLE permission_delegations (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    delegator_id    UUID         NOT NULL REFERENCES users(id),
    delegatee_id    UUID         NOT NULL REFERENCES users(id),
    permission_id   UUID         NOT NULL REFERENCES permissions(id),
    project_id      UUID         REFERENCES projects(id),
    reason          TEXT         NOT NULL,
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE'
                    CHECK (status IN ('ACTIVE', 'EXPIRED', 'REVOKED')),
    starts_at       TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ  NOT NULL,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    CHECK (expires_at > starts_at),
    CHECK (delegator_id != delegatee_id)
);

CREATE INDEX idx_delegations_delegator ON permission_delegations (delegator_id);
CREATE INDEX idx_delegations_delegatee ON permission_delegations (delegatee_id);
CREATE INDEX idx_delegations_status ON permission_delegations (status);
CREATE INDEX idx_delegations_expires ON permission_delegations (expires_at)
    WHERE status = 'ACTIVE';

-- ============================================================
-- 11. approval_requests
-- ============================================================
CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    request_type    VARCHAR(30)  NOT NULL
                    CHECK (request_type IN ('ROLE_ASSIGNMENT', 'PERMISSION_DELEGATION',
                           'CODE_CHANGE', 'BULK_IMPORT', 'CODE_DEPRECATION')),
    requester_id    UUID         NOT NULL REFERENCES users(id),
    title           VARCHAR(300) NOT NULL,
    description     TEXT,
    payload         JSONB        NOT NULL,         -- 승인 대상 데이터
    status          VARCHAR(20)  NOT NULL DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING', 'APPROVED', 'REJECTED', 'CANCELLED', 'EXPIRED')),
    project_id      UUID         REFERENCES projects(id),
    priority        VARCHAR(10)  NOT NULL DEFAULT 'NORMAL'
                    CHECK (priority IN ('LOW', 'NORMAL', 'HIGH', 'URGENT')),
    due_date        TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_approval_requests_requester ON approval_requests (requester_id);
CREATE INDEX idx_approval_requests_status ON approval_requests (status);
CREATE INDEX idx_approval_requests_type ON approval_requests (request_type);
CREATE INDEX idx_approval_requests_project ON approval_requests (project_id);
CREATE INDEX idx_approval_requests_created ON approval_requests (created_at DESC);

-- ============================================================
-- 12. approval_steps (다단계 승인)
-- ============================================================
CREATE TABLE approval_steps (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    request_id      UUID         NOT NULL REFERENCES approval_requests(id) ON DELETE CASCADE,
    step_order      INT          NOT NULL,
    approver_id     UUID         NOT NULL REFERENCES users(id),
    status          VARCHAR(20)  NOT NULL DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING', 'APPROVED', 'REJECTED', 'SKIPPED')),
    comment         TEXT,
    decided_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (request_id, step_order)
);

CREATE INDEX idx_approval_steps_request ON approval_steps (request_id);
CREATE INDEX idx_approval_steps_approver ON approval_steps (approver_id);
CREATE INDEX idx_approval_steps_status ON approval_steps (status);

-- ============================================================
-- 13. audit_logs (월별 파티션)
-- ============================================================
CREATE TABLE audit_logs (
    id              UUID         NOT NULL DEFAULT uuid_generate_v4(),
    actor_id        UUID         REFERENCES users(id),
    actor_ip        INET,
    actor_agent     VARCHAR(500),
    action          VARCHAR(50)  NOT NULL,
    resource_type   VARCHAR(50)  NOT NULL,
    resource_id     VARCHAR(255),
    project_id      UUID,
    details         JSONB,
    status          VARCHAR(10)  NOT NULL DEFAULT 'SUCCESS'
                    CHECK (status IN ('SUCCESS', 'FAILURE', 'ERROR')),
    error_message   TEXT,
    duration_ms     INT,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

-- 월별 파티션 생성 (예시: 2025년)
CREATE TABLE audit_logs_2025_01 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE audit_logs_2025_02 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
CREATE TABLE audit_logs_2025_03 PARTITION OF audit_logs
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');
-- ... (매월 파티션을 cron job 또는 pg_partman으로 자동 생성)

CREATE INDEX idx_audit_logs_actor ON audit_logs (actor_id, created_at DESC);
CREATE INDEX idx_audit_logs_action ON audit_logs (action, created_at DESC);
CREATE INDEX idx_audit_logs_resource ON audit_logs (resource_type, resource_id, created_at DESC);
CREATE INDEX idx_audit_logs_project ON audit_logs (project_id, created_at DESC);
CREATE INDEX idx_audit_logs_status ON audit_logs (status, created_at DESC);

-- ============================================================
-- 14. refresh_tokens
-- ============================================================
CREATE TABLE refresh_tokens (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash      VARCHAR(255) NOT NULL UNIQUE,  -- SHA-256 해시
    device_info     JSONB,
    ip_address      INET,
    expires_at      TIMESTAMPTZ  NOT NULL,
    is_revoked      BOOLEAN      NOT NULL DEFAULT FALSE,
    revoked_at      TIMESTAMPTZ,
    revoked_reason  VARCHAR(100),
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_refresh_tokens_user ON refresh_tokens (user_id);
CREATE INDEX idx_refresh_tokens_hash ON refresh_tokens (token_hash);
CREATE INDEX idx_refresh_tokens_expires ON refresh_tokens (expires_at)
    WHERE is_revoked = FALSE;

-- ============================================================
-- 15. sessions
-- ============================================================
CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID         NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    refresh_token_id UUID        REFERENCES refresh_tokens(id),
    device_fingerprint VARCHAR(255),
    ip_address      INET,
    user_agent      VARCHAR(500),
    is_active       BOOLEAN      NOT NULL DEFAULT TRUE,
    last_activity_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ  NOT NULL,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_sessions_user ON sessions (user_id);
CREATE INDEX idx_sessions_active ON sessions (user_id, is_active)
    WHERE is_active = TRUE;
CREATE INDEX idx_sessions_expires ON sessions (expires_at);
```

**Andrew Ng:** DDL이 꼼꼼합니다. `conditions JSONB`가 permissions 테이블에 있는 것이 좋네요. 나중에 AI 기반 동적 권한 평가에서도 활용할 수 있습니다.

**Martin Kleppmann:** 감사합니다. 이제 Master DB DDL입니다.

```sql
-- ============================================================
-- MASTER DATABASE DDL
-- PostgreSQL 16
-- ============================================================

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- ============================================================
-- 1. code_groups (코드 그룹)
-- ============================================================
CREATE TABLE code_groups (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    group_code      VARCHAR(50)  NOT NULL UNIQUE,
    parent_id       UUID         REFERENCES code_groups(id),
    depth           INT          NOT NULL DEFAULT 0,
    sort_order      INT          NOT NULL DEFAULT 0,
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE'
                    CHECK (status IN ('ACTIVE', 'INACTIVE', 'DEPRECATED')),
    owner_id        UUID         NOT NULL,         -- Auth DB users.id 참조
    data_type       VARCHAR(20)  NOT NULL DEFAULT 'STRING'
                    CHECK (data_type IN ('STRING', 'NUMBER', 'DATE', 'BOOLEAN')),
    max_code_length INT          NOT NULL DEFAULT 20,
    is_system       BOOLEAN      NOT NULL DEFAULT FALSE,  -- 시스템 코드 여부
    metadata        JSONB        NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_code_groups_code ON code_groups (group_code);
CREATE INDEX idx_code_groups_parent ON code_groups (parent_id);
CREATE INDEX idx_code_groups_status ON code_groups (status);
CREATE INDEX idx_code_groups_owner ON code_groups (owner_id);

-- ============================================================
-- 2. code_group_names (다국어 그룹명)
-- ============================================================
CREATE TABLE code_group_names (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    group_id        UUID         NOT NULL REFERENCES code_groups(id) ON DELETE CASCADE,
    locale          VARCHAR(10)  NOT NULL,         -- ko, en, ja, zh 등
    name            VARCHAR(200) NOT NULL,
    description     TEXT,
    UNIQUE (group_id, locale)
);

CREATE INDEX idx_code_group_names_group ON code_group_names (group_id);

-- ============================================================
-- 3. master_codes (마스터 코드)
-- ============================================================
CREATE TABLE master_codes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    group_id        UUID         NOT NULL REFERENCES code_groups(id),
    code            VARCHAR(50)  NOT NULL,
    parent_code_id  UUID         REFERENCES master_codes(id),
    sort_order      INT          NOT NULL DEFAULT 0,
    status          VARCHAR(20)  NOT NULL DEFAULT 'ACTIVE'
                    CHECK (status IN ('ACTIVE', 'INACTIVE', 'DEPRECATED', 'PENDING_APPROVAL')),
    effective_from  DATE,
    effective_to    DATE,
    version         INT          NOT NULL DEFAULT 1,
    is_system       BOOLEAN      NOT NULL DEFAULT FALSE,
    metadata        JSONB        NOT NULL DEFAULT '{}',
    created_by      UUID         NOT NULL,
    updated_by      UUID,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (group_id, code),
    CHECK (effective_to IS NULL OR effective_to >= effective_from)
);

CREATE INDEX idx_master_codes_group ON master_codes (group_id);
CREATE INDEX idx_master_codes_code ON master_codes (group_id, code);
CREATE INDEX idx_master_codes_status ON master_codes (status);
CREATE INDEX idx_master_codes_parent ON master_codes (parent_code_id);
CREATE INDEX idx_master_codes_effective ON master_codes (effective_from, effective_to)
    WHERE status = 'ACTIVE';

-- ============================================================
-- 4. master_code_names (다국어 코드명)
-- ============================================================
CREATE TABLE master_code_names (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code_id         UUID         NOT NULL REFERENCES master_codes(id) ON DELETE CASCADE,
    locale          VARCHAR(10)  NOT NULL,
    name            VARCHAR(200) NOT NULL,
    abbreviation    VARCHAR(50),
    description     TEXT,
    UNIQUE (code_id, locale)
);

CREATE INDEX idx_master_code_names_code ON master_code_names (code_id);

-- ============================================================
-- 5. master_code_attributes (확장 속성)
-- ============================================================
CREATE TABLE master_code_attributes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code_id         UUID         NOT NULL REFERENCES master_codes(id) ON DELETE CASCADE,
    attribute_key   VARCHAR(100) NOT NULL,
    attribute_value TEXT         NOT NULL,
    value_type      VARCHAR(20)  NOT NULL DEFAULT 'STRING'
                    CHECK (value_type IN ('STRING', 'NUMBER', 'BOOLEAN', 'JSON', 'DATE')),
    sort_order      INT          NOT NULL DEFAULT 0,
    UNIQUE (code_id, attribute_key)
);

CREATE INDEX idx_master_code_attrs_code ON master_code_attributes (code_id);
CREATE INDEX idx_master_code_attrs_key ON master_code_attributes (attribute_key);

-- ============================================================
-- 6. code_versions (변경 이력)
-- ============================================================
CREATE TABLE code_versions (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code_id         UUID         NOT NULL REFERENCES master_codes(id) ON DELETE CASCADE,
    version         INT          NOT NULL,
    change_type     VARCHAR(20)  NOT NULL
                    CHECK (change_type IN ('CREATE', 'UPDATE', 'DEPRECATE', 'ACTIVATE', 'DELETE')),
    before_data     JSONB,
    after_data      JSONB        NOT NULL,
    changed_by      UUID         NOT NULL,
    change_reason   TEXT,
    approval_id     UUID,                          -- Auth DB approval_requests.id
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (code_id, version)
);

CREATE INDEX idx_code_versions_code ON code_versions (code_id);
CREATE INDEX idx_code_versions_changed_by ON code_versions (changed_by);
CREATE INDEX idx_code_versions_created ON code_versions (created_at DESC);

-- ============================================================
-- 7. code_dependencies (코드 간 의존)
-- ============================================================
CREATE TABLE code_dependencies (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    source_code_id  UUID         NOT NULL REFERENCES master_codes(id) ON DELETE CASCADE,
    target_code_id  UUID         NOT NULL REFERENCES master_codes(id) ON DELETE CASCADE,
    dependency_type VARCHAR(30)  NOT NULL
                    CHECK (dependency_type IN ('REQUIRES', 'CONFLICTS', 'REPLACES', 'EXTENDS')),
    description     TEXT,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (source_code_id, target_code_id, dependency_type),
    CHECK (source_code_id != target_code_id)
);

CREATE INDEX idx_code_deps_source ON code_dependencies (source_code_id);
CREATE INDEX idx_code_deps_target ON code_dependencies (target_code_id);

-- ============================================================
-- 8. code_mappings (외부 시스템 매핑)
-- ============================================================
CREATE TABLE code_mappings (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    code_id         UUID         NOT NULL REFERENCES master_codes(id) ON DELETE CASCADE,
    external_system VARCHAR(50)  NOT NULL,         -- ERP, SAP, Legacy 등
    external_code   VARCHAR(100) NOT NULL,
    external_name   VARCHAR(200),
    mapping_note    TEXT,
    is_active       BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    UNIQUE (code_id, external_system)
);

CREATE INDEX idx_code_mappings_code ON code_mappings (code_id);
CREATE INDEX idx_code_mappings_external ON code_mappings (external_system, external_code);

-- ============================================================
-- 9. bulk_import_jobs (대량 업로드)
-- ============================================================
CREATE TABLE bulk_import_jobs (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    file_name       VARCHAR(255) NOT NULL,
    file_size       BIGINT       NOT NULL,
    file_hash       VARCHAR(64)  NOT NULL,          -- SHA-256
    group_id        UUID         NOT NULL REFERENCES code_groups(id),
    total_rows      INT          NOT NULL DEFAULT 0,
    success_count   INT          NOT NULL DEFAULT 0,
    error_count     INT          NOT NULL DEFAULT 0,
    skip_count      INT          NOT NULL DEFAULT 0,
    status          VARCHAR(20)  NOT NULL DEFAULT 'UPLOADED'
                    CHECK (status IN ('UPLOADED', 'VALIDATING', 'VALIDATED',
                           'PENDING_APPROVAL', 'APPLYING', 'COMPLETED', 'FAILED', 'CANCELLED')),
    validation_result JSONB,
    error_details   JSONB,
    uploaded_by     UUID         NOT NULL,
    approval_id     UUID,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_bulk_import_jobs_group ON bulk_import_jobs (group_id);
CREATE INDEX idx_bulk_import_jobs_status ON bulk_import_jobs (status);
CREATE INDEX idx_bulk_import_jobs_uploaded_by ON bulk_import_jobs (uploaded_by);

-- ============================================================
-- 10. outbox_events (Transactional Outbox)
-- ============================================================
CREATE TABLE outbox_events (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    aggregate_type  VARCHAR(50)  NOT NULL,         -- 'MasterCode', 'CodeGroup' 등
    aggregate_id    UUID         NOT NULL,
    event_type      VARCHAR(80)  NOT NULL,         -- 'CODE_CREATED', 'CODE_UPDATED' 등
    payload         JSONB        NOT NULL,
    metadata        JSONB        NOT NULL DEFAULT '{}',
    status          VARCHAR(20)  NOT NULL DEFAULT 'PENDING'
                    CHECK (status IN ('PENDING', 'PUBLISHED', 'FAILED')),
    retry_count     INT          NOT NULL DEFAULT 0,
    max_retries     INT          NOT NULL DEFAULT 3,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outbox_events_status ON outbox_events (status, created_at)
    WHERE status = 'PENDING';
CREATE INDEX idx_outbox_events_aggregate ON outbox_events (aggregate_type, aggregate_id);
```

**Martin Kleppmann:** 이제 공통 트리거들을 정의합니다. `updated_at` 자동 갱신과 `audit_logs` 불변성 보장 트리거입니다.

```sql
-- ============================================================
-- TRIGGERS
-- ============================================================

-- (A) updated_at 자동 갱신 트리거
CREATE OR REPLACE FUNCTION trigger_set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Auth DB 테이블에 적용
CREATE TRIGGER set_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON roles
    FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON resources
    FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON projects
    FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON approval_requests
    FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON permission_delegations
    FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();

-- Master DB 테이블에 적용
CREATE TRIGGER set_updated_at BEFORE UPDATE ON code_groups
    FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON master_codes
    FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON code_mappings
    FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();
CREATE TRIGGER set_updated_at BEFORE UPDATE ON bulk_import_jobs
    FOR EACH ROW EXECUTE FUNCTION trigger_set_updated_at();

-- (B) audit_logs 불변성 트리거 (UPDATE/DELETE 차단)
CREATE OR REPLACE FUNCTION trigger_prevent_audit_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_logs table is immutable. UPDATE and DELETE operations are forbidden.';
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER prevent_audit_update
    BEFORE UPDATE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION trigger_prevent_audit_mutation();

CREATE TRIGGER prevent_audit_delete
    BEFORE DELETE ON audit_logs
    FOR EACH ROW EXECUTE FUNCTION trigger_prevent_audit_mutation();
```

**Robert C. Martin:** 트리거가 깔끔합니다. audit_logs의 불변성이 코드 레벨이 아닌 DB 레벨에서 보장되는 점이 좋습니다. 어떤 ORM 버그가 있어도 감사 로그를 변조할 수 없죠.

**Bruce Schneier:** 보안 관점에서 매우 중요합니다. audit_logs에 DELETE를 실행하려면 `superuser`로 트리거를 비활성화해야 하므로, 그 자체가 또 다른 감사 이벤트가 됩니다. 운영 환경에서는 PostgreSQL의 `ALTER TABLE ... ENABLE ALWAYS TRIGGER`를 사용해서 superuser도 트리거를 건너뛸 수 없게 설정하는 것을 권장합니다.

**Martin Kleppmann:** 인덱스 전략을 요약하면:

| 테이블 | 핵심 인덱스 | 용도 |
|--------|-----------|------|
| `users` | `employee_id`, `email`, `idp_sub` | 로그인/조회 |
| `permissions` | `(resource_id, action)` UNIQUE | 권한 체크 |
| `role_permissions` | `role_id`, `permission_id` | JOIN 최적화 |
| `audit_logs` | `(actor_id, created_at DESC)`, `(resource_type, resource_id, created_at DESC)` | 감사 조회 |
| `master_codes` | `(group_id, code)` UNIQUE, `(effective_from, effective_to)` partial | 코드 조회 |
| `outbox_events` | `(status, created_at)` partial WHERE PENDING | 이벤트 발행 폴링 |

**Martin Fowler:** 파티셔닝 전략도 명확합니다. audit_logs는 월별 range partition으로 가고, `pg_partman`으로 자동 파티션 생성하면 운영 부담도 줄어듭니다.

**Kent Beck:** 완벽한 DDL입니다. 다음 토의로 넘어갑시다.

> **합의:** Auth DB 15개 테이블 + Master DB 10개 테이블 전체 DDL 확정. audit_logs는 월별 range partition 적용. 불변성 트리거와 updated_at 자동 갱신 트리거 포함. 모든 FK에 ON DELETE CASCADE/SET NULL 명시. outbox_events로 Transactional Outbox 패턴 지원.

---

## 토의 6: 시퀀스 다이어그램 — 주요 흐름 (7.6)

**Kent Beck:** 여섯 번째 주제, 시스템의 주요 6가지 흐름에 대한 시퀀스 다이어그램입니다. ASCII 아트로 표현해주세요.

**Martin Fowler:** 네, 6개 흐름 모두 상세히 그려보겠습니다.

### 흐름 1: SSO 로그인

```
┌────────┐     ┌──────────┐     ┌──────────────┐     ┌────────────┐
│Browser │     │Shell App │     │Auth Service  │     │Keycloak IdP│
└───┬────┘     └────┬─────┘     └──────┬───────┘     └─────┬──────┘
    │               │                   │                    │
    │  1. /login    │                   │                    │
    │──────────────>│                   │                    │
    │               │                   │                    │
    │               │ 2. GET /auth/sso/login                │
    │               │──────────────────>│                    │
    │               │                   │                    │
    │               │  3. 302 Redirect  │                    │
    │               │  (OIDC auth URL   │                    │
    │               │   + state + nonce)│                    │
    │               │<──────────────────│                    │
    │               │                   │                    │
    │  4. Redirect to Keycloak          │                    │
    │<──────────────│                   │                    │
    │               │                   │                    │
    │  5. Login Form (ID/PW + MFA)      │                    │
    │──────────────────────────────────────────────────────>│
    │               │                   │                    │
    │               │                   │     6. Validate    │
    │               │                   │     credentials    │
    │               │                   │                    │
    │  7. 302 Redirect + authorization_code                  │
    │<──────────────────────────────────────────────────────│
    │               │                   │                    │
    │  8. /auth/callback?code=xxx&state=yyy                  │
    │──────────────>│                   │                    │
    │               │                   │                    │
    │               │ 9. POST /auth/sso/callback             │
    │               │   (code, state)   │                    │
    │               │──────────────────>│                    │
    │               │                   │                    │
    │               │                   │ 10. Exchange code  │
    │               │                   │     for tokens     │
    │               │                   │───────────────────>│
    │               │                   │                    │
    │               │                   │ 11. id_token +     │
    │               │                   │     access_token   │
    │               │                   │<───────────────────│
    │               │                   │                    │
    │               │                   │ 12. Verify id_token│
    │               │                   │     Extract claims │
    │               │                   │     Upsert user    │
    │               │                   │     Generate JWT   │
    │               │                   │     Create session │
    │               │                   │                    │
    │               │ 13. { accessToken,│                    │
    │               │      refreshToken,│                    │
    │               │      user }       │                    │
    │               │<──────────────────│                    │
    │               │                   │                    │
    │  14. Store tokens                 │                    │
    │      (memory + httpOnly cookie)   │                    │
    │  15. Redirect to Dashboard        │                    │
    │<──────────────│                   │                    │
    │               │                   │                    │
```

### 흐름 2: 권한 체크

```
┌────────┐  ┌──────────┐  ┌──────────────┐  ┌───────────┐  ┌─────────┐
│Client  │  │Kong GW   │  │Auth Service  │  │OPA Sidecar│  │Service  │
└───┬────┘  └────┬─────┘  └──────┬───────┘  └─────┬─────┘  └────┬────┘
    │            │               │                 │              │
    │ 1. API Request             │                 │              │
    │   (Authorization: Bearer)  │                 │              │
    │───────────>│               │                 │              │
    │            │               │                 │              │
    │            │ 2. JWT 검증   │                 │              │
    │            │   (signature, │                 │              │
    │            │    expiry)    │                 │              │
    │            │──────────────>│                 │              │
    │            │               │                 │              │
    │            │ 3. Token Valid│                 │              │
    │            │   + user claims                 │              │
    │            │<──────────────│                 │              │
    │            │               │                 │              │
    │            │ 4. Forward request              │              │
    │            │   + X-User-Id │                 │              │
    │            │   + X-User-Roles                │              │
    │            │─────────────────────────────────────────────>│
    │            │               │                 │              │
    │            │               │                 │  5. Check    │
    │            │               │                 │  permission  │
    │            │               │                 │<─────────────│
    │            │               │                 │              │
    │            │               │  6. Evaluate    │              │
    │            │               │  Rego policy    │              │
    │            │               │  (cached rules) │              │
    │            │               │                 │              │
    │            │               │                 │  7. Allow/   │
    │            │               │                 │     Deny     │
    │            │               │                 │─────────────>│
    │            │               │                 │              │
    │            │               │                 │  [if Allow]  │
    │            │               │                 │  8. Process  │
    │            │               │                 │     request  │
    │            │               │                 │              │
    │ 9. Response (200 OK or 403 Forbidden)        │              │
    │<───────────────────────────────────────────────────────────│
    │            │               │                 │              │
```

### 흐름 3: 마스터 코드 조회 (캐시 계층)

```
┌──────────┐  ┌────────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐
│App       │  │Master SDK  │  │Local Cache │  │Redis      │  │MDM Svc   │
│(Consumer)│  │(@pkg/sdk)  │  │(Memory)    │  │(L2 Cache) │  │+ PG DB   │
└────┬─────┘  └─────┬──────┘  └─────┬─────┘  └─────┬─────┘  └────┬─────┘
     │              │               │               │              │
     │ 1. getCode   │               │               │              │
     │  ('DEPT',    │               │               │              │
     │   'D001')    │               │               │              │
     │─────────────>│               │               │              │
     │              │               │               │              │
     │              │ 2. L1 lookup  │               │              │
     │              │──────────────>│               │              │
     │              │               │               │              │
     │              │ 3a. [HIT]     │               │              │
     │              │   return data │               │              │
     │              │<──────────────│               │              │
     │<─────────────│ (TTL: 5min)  │               │              │
     │              │               │               │              │
     │              │ 3b. [MISS]    │               │              │
     │              │               │               │              │
     │              │ 4. L2 lookup  │               │              │
     │              │──────────────────────────────>│              │
     │              │               │               │              │
     │              │ 5a. [HIT]     │               │              │
     │              │   return data │               │              │
     │              │<──────────────────────────────│              │
     │              │               │               │              │
     │              │ 5b. [MISS]    │               │              │
     │              │               │               │              │
     │              │ 6. GET /api/v1/codes/DEPT/D001│              │
     │              │────────────────────────────────────────────>│
     │              │               │               │              │
     │              │               │               │  7. SELECT   │
     │              │               │               │  FROM master │
     │              │               │               │  _codes ...  │
     │              │               │               │              │
     │              │ 8. Response { code, names, attrs }          │
     │              │<────────────────────────────────────────────│
     │              │               │               │              │
     │              │ 9. SET Redis  │               │              │
     │              │  (TTL: 30min) │               │              │
     │              │──────────────────────────────>│              │
     │              │               │               │              │
     │              │ 10. SET L1    │               │              │
     │              │  (TTL: 5min)  │               │              │
     │              │──────────────>│               │              │
     │              │               │               │              │
     │ 11. return   │               │               │              │
     │     data     │               │               │              │
     │<─────────────│               │               │              │
     │              │               │               │              │
```

### 흐름 4: 마스터 코드 변경 승인

```
┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────┐
│Requester │  │MDM API   │  │Approval Svc  │  │Kafka     │  │Subscriber│
│(Browser) │  │Service   │  │(Auth Domain) │  │          │  │Services  │
└────┬─────┘  └────┬─────┘  └──────┬───────┘  └────┬─────┘  └────┬─────┘
     │              │               │                │              │
     │ 1. POST      │               │                │              │
     │ /codes       │               │                │              │
     │ (변경 요청)  │               │                │              │
     │─────────────>│               │                │              │
     │              │               │                │              │
     │              │ 2. Validate   │                │              │
     │              │    payload    │                │              │
     │              │    Save as    │                │              │
     │              │    PENDING    │                │              │
     │              │               │                │              │
     │              │ 3. POST /approval-requests     │              │
     │              │   { type: CODE_CHANGE,         │              │
     │              │     payload: {...} }            │              │
     │              │──────────────>│                │              │
     │              │               │                │              │
     │              │               │ 4. Create      │              │
     │              │               │    request +   │              │
     │              │               │    steps       │              │
     │              │               │                │              │
     │              │               │ 5. Send        │              │
     │              │               │   notification │              │
     │              │               │   to approver  │              │
     │              │               │   (Email/Slack)│              │
     │              │               │                │              │
     │ 6. 201 Created               │                │              │
     │   { requestId, status }      │                │              │
     │<─────────────│               │                │              │
     │              │               │                │              │
     ═══════════════════════════════════════════════════════════════
          (시간 경과: Data Owner가 승인 화면 접근)
     ═══════════════════════════════════════════════════════════════
     │              │               │                │              │
     │              │  7. POST /approval-requests     │              │
     │              │    /{id}/steps/{order}/approve  │              │
     │              │──────────────>│                │              │
     │              │               │                │              │
     │              │               │ 8. Update step │              │
     │              │               │    status =    │              │
     │              │               │    APPROVED    │              │
     │              │               │    Check all   │              │
     │              │               │    steps done  │              │
     │              │               │                │              │
     │              │  9. Callback: │                │              │
     │              │  approval     │                │              │
     │              │  completed    │                │              │
     │              │<──────────────│                │              │
     │              │               │                │              │
     │              │ 10. Apply     │                │              │
     │              │     change    │                │              │
     │              │  (UPDATE code │                │              │
     │              │   + INSERT    │                │              │
     │              │   version +   │                │              │
     │              │   INSERT      │                │              │
     │              │   outbox)     │                │              │
     │              │               │                │              │
     │              │ 11. Outbox    │                │              │
     │              │   poller      │                │              │
     │              │   publishes   │                │              │
     │              │──────────────────────────────>│              │
     │              │               │                │              │
     │              │               │   12. Consume  │              │
     │              │               │   CODE_UPDATED │              │
     │              │               │                │─────────────>│
     │              │               │                │              │
     │              │               │                │  13. Update  │
     │              │               │                │  local cache │
     │              │               │                │  & data      │
     │              │               │                │              │
```

### 흐름 5: Micro Frontend 앱 로딩

```
┌────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────┐  ┌──────────┐
│Browser │  │Shell App │  │Registry API  │  │CDN       │  │Remote App│
└───┬────┘  └────┬─────┘  └──────┬───────┘  └────┬─────┘  └────┬─────┘
    │            │               │                │              │
    │ 1. Navigate│               │                │              │
    │   to /app  │               │                │              │
    │───────────>│               │                │              │
    │            │               │                │              │
    │            │ 2. GET /registry/apps           │              │
    │            │──────────────>│                │              │
    │            │               │                │              │
    │            │ 3. App manifests                │              │
    │            │   [{ name, url,│                │              │
    │            │     version,   │                │              │
    │            │     permissions│                │              │
    │            │   }]           │                │              │
    │            │<──────────────│                │              │
    │            │               │                │              │
    │            │ 4. Check user │                │              │
    │            │    permission │                │              │
    │            │    for app    │                │              │
    │            │               │                │              │
    │            │ 5. Load remoteEntry.js          │              │
    │            │────────────────────────────────>│              │
    │            │               │                │              │
    │            │ 6. JS bundle  │                │              │
    │            │<────────────────────────────────│              │
    │            │               │                │              │
    │            │ 7. import('./RemoteApp')        │              │
    │            │               │                │              │
    │            │ 8. mount(container, {           │              │
    │            │      user, token,               │              │
    │            │      navigate,                  │              │
    │            │      eventBus,                  │              │
    │            │      masterCodeSDK              │              │
    │            │    })          │                │              │
    │            │────────────────────────────────────────────>│
    │            │               │                │              │
    │            │               │                │  9. init()   │
    │            │               │                │  render UI   │
    │            │               │                │              │
    │ 10. App rendered in shell container          │              │
    │<───────────────────────────────────────────────────────────│
    │            │               │                │              │
```

### 흐름 6: Token 갱신

```
┌────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────────┐
│App Code│  │Auth SDK  │  │API Gateway   │  │Auth Service  │
│        │  │(Interceptor)│(Kong)        │  │              │
└───┬────┘  └────┬─────┘  └──────┬───────┘  └──────┬───────┘
    │            │               │                  │
    │ 1. API call│               │                  │
    │───────────>│               │                  │
    │            │               │                  │
    │            │ 2. Attach     │                  │
    │            │    Access     │                  │
    │            │    Token      │                  │
    │            │──────────────>│                  │
    │            │               │                  │
    │            │               │ 3. Verify JWT    │
    │            │               │    → EXPIRED!    │
    │            │               │                  │
    │            │ 4. 401        │                  │
    │            │   Unauthorized│                  │
    │            │   (TOKEN_     │                  │
    │            │    EXPIRED)   │                  │
    │            │<──────────────│                  │
    │            │               │                  │
    │            │ 5. Intercept 401                 │
    │            │    Check: is token               │
    │            │    refresh in progress?          │
    │            │    → Queue request               │
    │            │               │                  │
    │            │ 6. POST /auth/refresh            │
    │            │   { refreshToken }               │
    │            │──────────────────────────────────>│
    │            │               │                  │
    │            │               │  7. Validate     │
    │            │               │     refresh token│
    │            │               │     (DB lookup + │
    │            │               │      rotation)   │
    │            │               │                  │
    │            │ 8. New tokens │                  │
    │            │   { accessToken,                 │
    │            │     refreshToken }               │
    │            │<──────────────────────────────────│
    │            │               │                  │
    │            │ 9. Update     │                  │
    │            │    stored     │                  │
    │            │    tokens     │                  │
    │            │               │                  │
    │            │ 10. Retry original request       │
    │            │    with new Access Token         │
    │            │──────────────>│                  │
    │            │               │                  │
    │            │               │ 11. Verify JWT   │
    │            │               │     → VALID      │
    │            │               │     Forward      │
    │            │               │                  │
    │ 12. Original               │                  │
    │     response               │                  │
    │<───────────│               │                  │
    │            │               │                  │
    │            │ 13. Dequeue & retry              │
    │            │     pending requests             │
    │            │               │                  │
```

**Andrej Karpathy:** 흐름 3의 캐시 계층이 잘 설계되었습니다. L1(메모리 5분) → L2(Redis 30분) → DB 구조로 가면, 대부분의 코드 조회는 L1에서 해결됩니다.

**Robert C. Martin:** 흐름 6의 Token 갱신에서 Queue 패턴이 중요합니다. 동시에 여러 API 호출이 401을 받았을 때, 하나의 refresh만 실행하고 나머지는 대기시키는 패턴이죠. 실제 코드에서는 Promise 기반 Mutex를 사용합니다.

**Bruce Schneier:** 흐름 2에서 OPA sidecar가 서비스 옆에 위치하는 것이 보안 아키텍처상 올바릅니다. Gateway에서 인증(Authentication)을 하고, 서비스 레벨에서 인가(Authorization)를 하는 이중 검증 구조입니다.

**Jakob Nielsen:** 흐름 5의 Micro Frontend 로딩에서 permission check가 4번째 단계에 있는 것이 좋습니다. 사용자가 접근 권한이 없는 앱의 JS를 불필요하게 다운로드하지 않도록 사전에 체크하니까요.

> **합의:** 6개 주요 흐름 시퀀스 다이어그램 확정. SSO는 OIDC Authorization Code Flow, 권한 체크는 Gateway 인증 + OPA 인가 이중 구조, 캐시는 L1(5분)/L2(30분)/DB 3계층, 변경 승인은 다단계 + Outbox + Kafka, MFE는 Registry 기반 동적 로딩, Token 갱신은 Queue 기반 단일 refresh.

---

## 토의 7: 환경 설정 — 의존성, 환경변수, Docker Compose (7.7)

**Kent Beck:** 일곱 번째 주제입니다. 개발 환경을 일관성 있게 구성하기 위한 기술 스택 버전, 환경변수, Docker Compose 설정을 확정합시다.

**Robert C. Martin:** 먼저 전체 기술 스택 버전 매트릭스를 정리하겠습니다. 모든 팀이 동일한 버전을 사용해야 "내 PC에서는 되는데" 문제를 방지합니다.

```yaml
# ============================================================
# TECH STACK - Version Matrix
# ============================================================

# --- Runtime & Language ---
node:           "20.11.x LTS"       # .nvmrc에 고정
typescript:     "5.4.x"              # strict mode 필수
pnpm:           "8.15.x"            # corepack enable로 강제

# --- Frontend ---
react:          "18.3.x"
react-dom:      "18.3.x"
vite:           "5.2.x"
tailwindcss:    "3.4.x"
zustand:        "4.5.x"             # 상태 관리
react-query:    "5.x"               # @tanstack/react-query
react-router:   "6.22.x"
react-hook-form: "7.51.x"
zod:            "3.22.x"            # 스키마 검증
i18next:        "23.x"              # 다국어
module-federation: "webpack/enhanced" # Vite plugin

# --- Backend ---
nestjs:         "10.3.x"            # @nestjs/core
typeorm:        "0.3.20"
class-validator: "0.14.x"
class-transformer: "0.5.x"
passport:       "0.7.x"
passport-jwt:   "4.0.x"
ioredis:        "5.3.x"
kafkajs:        "2.2.x"
bull:           "5.x"               # @nestjs/bullmq (Job Queue)

# --- Database & Messaging ---
postgresql:     "16.2"
redis:          "7.2.x"
elasticsearch:  "8.12.x"
kafka:          "3.6.x"             # Apache Kafka (KRaft mode)
zookeeper:      "N/A"               # KRaft 모드이므로 불필요

# --- Infrastructure ---
docker:         "25.x"
kubernetes:     "1.29.x"
terraform:      "1.7.x"
vault:          "1.15.x"            # HashiCorp Vault
kong:           "3.6.x"             # API Gateway
keycloak:       "24.x"              # Identity Provider
opa:            "0.62.x"            # Open Policy Agent

# --- Observability ---
opentelemetry-sdk: "1.22.x"         # @opentelemetry/sdk-node
grafana:        "10.3.x"
loki:           "2.9.x"             # 로그 수집
tempo:          "2.3.x"             # 분산 트레이싱
prometheus:     "2.50.x"            # 메트릭 수집

# --- Testing ---
vitest:         "1.3.x"             # Unit / Integration
playwright:     "1.41.x"            # E2E
pact:           "12.x"              # @pact-foundation/pact
k6:             "0.48.x"            # 성능 테스트
msw:            "2.2.x"             # API Mocking (Mock Service Worker)

# --- Build & CI ---
turborepo:      "1.12.x"
github-actions: "latest"
docker-compose: "2.24.x"
```

**Martin Fowler:** 좋습니다. 이제 환경변수 파일을 서비스별로 정리합시다.

```bash
# ============================================================
# .env.example — 공통 환경변수
# ============================================================

# --- Common ---
NODE_ENV=development                     # development | staging | production
LOG_LEVEL=debug                          # debug | info | warn | error
LOG_FORMAT=json                          # json | pretty
PORT=3000                                # 서비스 포트 (서비스별 다름)
SERVICE_NAME=auth-service                # 서비스 식별자
CORS_ORIGINS=http://localhost:5173       # 허용 Origin (comma-separated)

# --- Auth / JWT ---
JWT_SECRET=your-256-bit-secret-key-here  # HS256 (개발용, 운영은 Vault)
JWT_ACCESS_TTL=900                       # Access Token TTL (초): 15분
JWT_REFRESH_TTL=604800                   # Refresh Token TTL (초): 7일
JWT_ISSUER=mdm-platform                  # JWT issuer claim
JWT_AUDIENCE=mdm-api                     # JWT audience claim

# --- OIDC / Keycloak ---
OIDC_ISSUER=http://localhost:8080/realms/mdm
OIDC_CLIENT_ID=mdm-web-client
OIDC_CLIENT_SECRET=your-oidc-client-secret
OIDC_REDIRECT_URI=http://localhost:5173/auth/callback
OIDC_LOGOUT_URI=http://localhost:5173/auth/logout
SESSION_MAX_CONCURRENT=3                 # 동시 세션 제한

# --- Database (PostgreSQL) ---
DATABASE_URL=postgresql://mdm_user:mdm_pass@localhost:5432/mdm_auth
DATABASE_POOL_SIZE=10                    # TypeORM pool size
DATABASE_SSL=false                       # 운영 환경에서는 true
DATABASE_LOG_QUERIES=true                # 개발시 SQL 로깅
DATABASE_MASTER_URL=postgresql://mdm_user:mdm_pass@localhost:5432/mdm_master

# --- Redis ---
REDIS_URL=redis://localhost:6379/0
REDIS_PASSWORD=                          # 개발 환경은 비밀번호 없음
REDIS_KEY_PREFIX=mdm:                    # 키 네임스페이스
REDIS_CACHE_TTL=1800                     # 기본 TTL (초): 30분

# --- Kafka ---
KAFKA_BROKERS=localhost:9092             # comma-separated broker list
KAFKA_CLIENT_ID=mdm-service
KAFKA_GROUP_ID=mdm-consumer-group
KAFKA_SASL_ENABLED=false                 # 운영은 true
KAFKA_SASL_MECHANISM=SCRAM-SHA-256
KAFKA_SASL_USERNAME=
KAFKA_SASL_PASSWORD=

# --- Elasticsearch ---
ELASTICSEARCH_URL=http://localhost:9200
ELASTICSEARCH_INDEX_PREFIX=mdm_          # 인덱스 접두사
ELASTICSEARCH_USERNAME=elastic
ELASTICSEARCH_PASSWORD=changeme

# --- HashiCorp Vault ---
VAULT_ADDR=http://localhost:8200
VAULT_TOKEN=dev-root-token               # 개발용 root token
VAULT_SECRET_PATH=secret/data/mdm        # 시크릿 경로
VAULT_ENABLED=false                      # 개발 환경에서는 비활성

# --- Feature Flags ---
FEATURE_AI_SEARCH=false                  # AI 기반 코드 검색
FEATURE_MFA_REQUIRED=false               # MFA 강제 활성화
FEATURE_BULK_IMPORT=true                 # Excel 대량 업로드
FEATURE_AUDIT_EXPORT=true                # 감사 로그 내보내기
FEATURE_CODE_DEPENDENCIES=false          # 코드 의존성 관리

# --- Observability ---
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_SERVICE_NAME=${SERVICE_NAME}
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1             # 운영: 10% 샘플링
```

**Martin Kleppmann:** Docker Compose 설정도 만들어야 합니다. 개발자가 `docker compose up`만 하면 전체 인프라가 뜨도록요.

```yaml
# ============================================================
# docker-compose.dev.yml — 로컬 개발 환경
# ============================================================
version: "3.9"

services:
  # --- PostgreSQL ---
  postgres:
    image: postgres:16.2-alpine
    container_name: mdm-postgres
    environment:
      POSTGRES_USER: mdm_user
      POSTGRES_PASSWORD: mdm_pass
      POSTGRES_DB: mdm_auth
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./infra/db/init:/docker-entrypoint-initdb.d  # 초기 DDL
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mdm_user"]
      interval: 5s
      timeout: 5s
      retries: 5

  # --- Redis ---
  redis:
    image: redis:7.2-alpine
    container_name: mdm-redis
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  # --- Kafka (KRaft mode) ---
  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: mdm-kafka
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_LOG_RETENTION_HOURS: 168
    ports:
      - "9092:9092"
    volumes:
      - kafka_data:/var/lib/kafka/data
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 10s
      timeout: 10s
      retries: 5

  # --- Kafka UI ---
  kafka-ui:
    image: provectuslabs/kafka-ui:v0.7.1
    container_name: mdm-kafka-ui
    environment:
      KAFKA_CLUSTERS_0_NAME: mdm-local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
    ports:
      - "8081:8080"
    depends_on:
      kafka:
        condition: service_healthy

  # --- Elasticsearch ---
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.12.2
    container_name: mdm-elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 10s
      timeout: 10s
      retries: 5

  # --- Keycloak (Identity Provider) ---
  keycloak:
    image: quay.io/keycloak/keycloak:24.0
    container_name: mdm-keycloak
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_DB: postgres
      KC_DB_URL_HOST: postgres
      KC_DB_URL_DATABASE: mdm_auth
      KC_DB_USERNAME: mdm_user
      KC_DB_PASSWORD: mdm_pass
    ports:
      - "8080:8080"
    command: start-dev --import-realm
    volumes:
      - ./infra/keycloak/realm-export.json:/opt/keycloak/data/import/realm.json
    depends_on:
      postgres:
        condition: service_healthy

  # --- Vault (Secret Management) ---
  vault:
    image: hashicorp/vault:1.15
    container_name: mdm-vault
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: dev-root-token
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    ports:
      - "8200:8200"
    cap_add:
      - IPC_LOCK

  # --- Grafana (Monitoring) ---
  grafana:
    image: grafana/grafana:10.3.3
    container_name: mdm-grafana
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
    ports:
      - "3100:3000"
    volumes:
      - ./infra/grafana/provisioning:/etc/grafana/provisioning
      - grafana_data:/var/lib/grafana

  # --- Loki (Log Aggregation) ---
  loki:
    image: grafana/loki:2.9.4
    container_name: mdm-loki
    ports:
      - "3101:3100"
    command: -config.file=/etc/loki/local-config.yaml

  # --- OTel Collector ---
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.95.0
    container_name: mdm-otel-collector
    ports:
      - "4317:4317"    # gRPC
      - "4318:4318"    # HTTP
    volumes:
      - ./infra/otel/otel-collector-config.yaml:/etc/otelcol-contrib/config.yaml

volumes:
  postgres_data:
  redis_data:
  kafka_data:
  es_data:
  grafana_data:
```

**Andrej Karpathy:** Feature flag 환경변수가 깔끔하게 정리되었습니다. `FEATURE_AI_SEARCH`는 나중에 AI 검색 모델이 준비되면 켜면 되겠네요.

**Bruce Schneier:** 보안 관점에서 몇 가지 강조합니다. 첫째, `JWT_SECRET`은 개발용으로만 `.env`에 두고, 운영에서는 반드시 Vault에서 읽어야 합니다. 둘째, `OIDC_CLIENT_SECRET`도 마찬가지입니다. Docker Compose에서 Vault 컨테이너를 포함한 것이 좋습니다. 셋째, `.env` 파일은 `.gitignore`에 반드시 포함되어야 합니다.

**Donald Knuth:** 환경변수 문서화가 잘 되어 있습니다. 주석으로 각 변수의 용도, 기본값, 허용 범위를 명시한 것이 유지보수에 큰 도움이 됩니다. `.env.example` 파일은 반드시 Git에 포함하되, 실제 `.env`는 `.gitignore`에 추가하는 이중 전략이 올바릅니다.

**Kent Beck:** Docker Compose에 healthcheck가 모든 서비스에 정의되어 있어서, `depends_on: condition: service_healthy`로 기동 순서를 제어할 수 있습니다.

> **합의:** 기술 스택 전체 버전 매트릭스 확정. 환경변수는 `.env.example`로 문서화하고 6개 그룹(Common, Auth, DB, Kafka, ES, Vault, Feature Flags, Observability)으로 분류. Docker Compose는 10개 서비스(PostgreSQL, Redis, Kafka, Kafka UI, Elasticsearch, Keycloak, Vault, Grafana, Loki, OTel Collector) 포함. 운영 시크릿은 Vault로 관리.

---

## 토의 8: 테스트 시나리오 (7.8)

**Kent Beck:** 마지막 여덟 번째 주제입니다. 테스트 전략과 시나리오를 확정합시다. 제가 XP(Extreme Programming)의 창시자로서 강조하지만, 테스트 없는 코드는 레거시입니다.

**Robert C. Martin:** 동의합니다. 먼저 커버리지 목표부터 설정하고, 각 레벨별 시나리오를 정의합시다.

### 커버리지 목표

| 테스트 레벨 | 도구 | 목표 커버리지 |
|------------|------|-------------|
| Unit Test | Vitest | 80% (line) |
| Integration Test | Vitest + Testcontainers | 70% (branch) |
| Contract Test | Pact | 핵심 API 100% |
| E2E Test | Playwright | Critical Path 100% |
| Performance Test | k6 | P95 < 200ms, Error < 0.1% |

### Unit Test 시나리오 (Vitest)

```typescript
// ============================================================
// 1. Permission Check Logic
// ============================================================
describe('PermissionService', () => {
  describe('checkPermission', () => {
    it('시스템 관리자는 모든 리소스에 접근 가능', async () => {
      const user = createMockUser({ systemRoles: ['SYSTEM_ADMIN'] });
      const result = await permissionService.check(user, 'mdm:codes', 'DELETE');
      expect(result.allowed).toBe(true);
      expect(result.reason).toBe('SYSTEM_ADMIN_BYPASS');
    });

    it('프로젝트 역할에 해당 권한이 있으면 허용', async () => {
      const user = createMockUser({
        projectRoles: [{ projectId: 'P1', roleCode: 'DATA_MANAGER' }]
      });
      mockRolePermissions('DATA_MANAGER', ['mdm:codes:READ', 'mdm:codes:UPDATE']);

      const result = await permissionService.check(user, 'mdm:codes', 'UPDATE', 'P1');
      expect(result.allowed).toBe(true);
    });

    it('권한 위임(delegation)이 유효 기간 내이면 허용', async () => {
      const delegation = createMockDelegation({
        status: 'ACTIVE',
        startsAt: subDays(new Date(), 1),
        expiresAt: addDays(new Date(), 5),
      });
      mockDelegation(delegation);

      const result = await permissionService.check(user, 'mdm:codes', 'APPROVE', 'P1');
      expect(result.allowed).toBe(true);
      expect(result.via).toBe('DELEGATION');
    });

    it('만료된 위임은 거부', async () => {
      const delegation = createMockDelegation({
        status: 'ACTIVE',
        expiresAt: subDays(new Date(), 1),   // 어제 만료
      });
      mockDelegation(delegation);

      const result = await permissionService.check(user, 'mdm:codes', 'APPROVE', 'P1');
      expect(result.allowed).toBe(false);
    });

    it('조건부 권한: department=same 조건이면 같은 부서만 허용', async () => {
      const user = createMockUser({ department: 'DEV' });
      const resource = createMockResource({ department: 'DEV' });
      mockPermissionCondition({ department: 'same' });

      const result = await permissionService.check(user, resource.id, 'READ');
      expect(result.allowed).toBe(true);
    });
  });
});

// ============================================================
// 2. Token Validation
// ============================================================
describe('TokenService', () => {
  it('유효한 JWT를 올바르게 검증', async () => {
    const token = tokenService.generateAccess({ userId: 'U1', roles: ['VIEWER'] });
    const decoded = await tokenService.verifyAccess(token);
    expect(decoded.userId).toBe('U1');
    expect(decoded.roles).toContain('VIEWER');
  });

  it('만료된 JWT는 TokenExpiredError 발생', async () => {
    const token = tokenService.generateAccess({ userId: 'U1' }, { expiresIn: '0s' });
    await expect(tokenService.verifyAccess(token))
      .rejects.toThrow('TOKEN_EXPIRED');
  });

  it('변조된 JWT는 InvalidTokenError 발생', async () => {
    const token = tokenService.generateAccess({ userId: 'U1' });
    const tampered = token.slice(0, -5) + 'xxxxx';
    await expect(tokenService.verifyAccess(tampered))
      .rejects.toThrow('INVALID_TOKEN');
  });

  it('Refresh Token rotation: 사용된 토큰은 재사용 불가', async () => {
    const { refreshToken } = await tokenService.generateTokenPair('U1');
    await tokenService.refresh(refreshToken);  // 첫 번째 사용
    await expect(tokenService.refresh(refreshToken))
      .rejects.toThrow('TOKEN_ALREADY_USED');
  });
});

// ============================================================
// 3. Cache Fallback
// ============================================================
describe('MasterCodeCache', () => {
  it('L1 캐시 히트 시 Redis를 호출하지 않음', async () => {
    l1Cache.set('DEPT:D001', mockCode);
    const result = await cacheService.get('DEPT', 'D001');
    expect(result).toEqual(mockCode);
    expect(redisClient.get).not.toHaveBeenCalled();
  });

  it('L1 미스, L2 히트 시 L1에 저장 후 반환', async () => {
    redisClient.get.mockResolvedValue(JSON.stringify(mockCode));
    const result = await cacheService.get('DEPT', 'D001');
    expect(result).toEqual(mockCode);
    expect(l1Cache.get('DEPT:D001')).toEqual(mockCode);
  });

  it('L1, L2 모두 미스 시 DB 조회 후 양쪽 캐시 저장', async () => {
    redisClient.get.mockResolvedValue(null);
    codeRepository.findOne.mockResolvedValue(mockCode);

    const result = await cacheService.get('DEPT', 'D001');
    expect(codeRepository.findOne).toHaveBeenCalled();
    expect(redisClient.setex).toHaveBeenCalledWith('mdm:codes:DEPT:D001', 1800, expect.any(String));
    expect(l1Cache.get('DEPT:D001')).toEqual(mockCode);
  });

  it('Redis 장애 시 DB 직접 조회로 fallback', async () => {
    redisClient.get.mockRejectedValue(new Error('ECONNREFUSED'));
    codeRepository.findOne.mockResolvedValue(mockCode);

    const result = await cacheService.get('DEPT', 'D001');
    expect(result).toEqual(mockCode);  // 정상 반환
  });
});

// ============================================================
// 4. Code Lifecycle
// ============================================================
describe('MasterCodeService', () => {
  it('코드 생성 시 version=1로 시작하고 이력 저장', async () => {
    const created = await codeService.create(createCodeDto);
    expect(created.version).toBe(1);
    expect(codeVersionRepo.save).toHaveBeenCalledWith(
      expect.objectContaining({ version: 1, changeType: 'CREATE' })
    );
  });

  it('DEPRECATED 상태의 코드는 업데이트 불가', async () => {
    mockExistingCode({ status: 'DEPRECATED' });
    await expect(codeService.update('C1', updateDto))
      .rejects.toThrow('CANNOT_UPDATE_DEPRECATED_CODE');
  });

  it('코드 폐기 시 의존 코드가 있으면 경고 반환', async () => {
    mockDependencies([{ targetCodeId: 'C1', type: 'REQUIRES' }]);
    const result = await codeService.deprecate('C1');
    expect(result.warnings).toContainEqual(
      expect.objectContaining({ type: 'HAS_DEPENDENTS' })
    );
  });
});
```

### Contract Test 시나리오 (Pact)

```typescript
// ============================================================
// Consumer: Auth SDK <-> Provider: Auth Service
// ============================================================
describe('Auth SDK -> Auth Service Contract', () => {
  const provider = new PactV4({
    consumer: 'AuthSDK',
    provider: 'AuthService',
  });

  it('POST /auth/login - 유효한 자격증명으로 토큰 반환', async () => {
    await provider
      .addInteraction()
      .given('사용자 john@example.com이 존재함')
      .uponReceiving('유효한 로그인 요청')
      .withRequest('POST', '/api/v1/auth/login', (builder) => {
        builder.jsonBody({ email: 'john@example.com', password: 'ValidPass1!' });
      })
      .willRespondWith(200, (builder) => {
        builder.jsonBody({
          data: {
            accessToken: like('eyJhbGciOiJIUzI1NiIs...'),
            refreshToken: like('dGhpcyBpcyBhIHJlZnJlc2g...'),
            expiresIn: integer(900),
            user: {
              id: uuid(),
              email: string('john@example.com'),
              name: string('John'),
            }
          }
        });
      })
      .executeTest(async (mockServer) => {
        const sdk = new AuthSDK({ baseUrl: mockServer.url });
        const result = await sdk.login('john@example.com', 'ValidPass1!');
        expect(result.data.accessToken).toBeDefined();
      });
  });

  it('POST /auth/refresh - 유효한 리프레시 토큰으로 갱신', async () => {
    await provider
      .addInteraction()
      .given('유효한 refresh token이 존재함')
      .uponReceiving('토큰 갱신 요청')
      .withRequest('POST', '/api/v1/auth/refresh', (builder) => {
        builder.jsonBody({ refreshToken: like('valid-refresh-token') });
      })
      .willRespondWith(200, (builder) => {
        builder.jsonBody({
          data: {
            accessToken: like('new-access-token'),
            refreshToken: like('new-refresh-token'),
            expiresIn: integer(900),
          }
        });
      })
      .executeTest(async (mockServer) => {
        const sdk = new AuthSDK({ baseUrl: mockServer.url });
        const result = await sdk.refresh('valid-refresh-token');
        expect(result.data.accessToken).toBeDefined();
      });
  });
});

// ============================================================
// Consumer: Master SDK <-> Provider: MDM Service
// ============================================================
describe('Master SDK -> MDM Service Contract', () => {
  it('GET /codes/:groupCode - 코드 그룹 조회', async () => {
    await provider
      .addInteraction()
      .given('DEPT 그룹에 3개 코드가 존재함')
      .uponReceiving('코드 그룹 조회 요청')
      .withRequest('GET', '/api/v1/codes/DEPT')
      .willRespondWith(200, (builder) => {
        builder.jsonBody({
          data: eachLike({
            id: uuid(),
            code: string('D001'),
            names: eachLike({ locale: string('ko'), name: string('개발팀') }),
            status: string('ACTIVE'),
          }),
          meta: { total: integer(3) }
        });
      })
      .executeTest(async (mockServer) => {
        const sdk = new MasterCodeSDK({ baseUrl: mockServer.url });
        const result = await sdk.getCodesByGroup('DEPT');
        expect(result.data.length).toBeGreaterThan(0);
      });
  });
});
```

### Integration Test 시나리오

```typescript
// ============================================================
// Auth + Permission 통합 테스트 (Testcontainers)
// ============================================================
describe('Auth + Permission Integration', () => {
  let pgContainer: StartedTestContainer;
  let redisContainer: StartedTestContainer;

  beforeAll(async () => {
    pgContainer = await new PostgreSqlContainer('postgres:16').start();
    redisContainer = await new GenericContainer('redis:7.2').withExposedPorts(6379).start();
    await runMigrations(pgContainer.getConnectionUri());
  });

  it('사용자 생성 → 역할 할당 → 권한 체크 전체 흐름', async () => {
    // 1. 사용자 생성
    const user = await userService.create({ email: 'test@test.com', name: 'Test' });

    // 2. 역할 생성 + 권한 부여
    const role = await roleService.create({ code: 'EDITOR', scope: 'PROJECT' });
    await permissionService.grantToRole(role.id, 'mdm:codes', 'UPDATE');

    // 3. 프로젝트 + 멤버 + 역할 할당
    const project = await projectService.create({ code: 'P1', ownerId: user.id });
    await projectService.addMember(project.id, user.id);
    await projectService.assignRole(project.id, user.id, role.id);

    // 4. 권한 체크
    const allowed = await permissionService.check(user.id, 'mdm:codes', 'UPDATE', project.id);
    expect(allowed).toBe(true);

    // 5. 다른 프로젝트는 거부
    const denied = await permissionService.check(user.id, 'mdm:codes', 'UPDATE', 'OTHER_PROJECT');
    expect(denied).toBe(false);
  });
});

// ============================================================
// MDM + Kafka 이벤트 전파 통합 테스트
// ============================================================
describe('MDM + Kafka Event Propagation', () => {
  it('코드 변경 → Outbox → Kafka → Consumer 수신 확인', async () => {
    // 1. 코드 변경
    const code = await codeService.update('C1', { status: 'ACTIVE' });

    // 2. Outbox에 이벤트가 저장되었는지 확인
    const outboxEvent = await outboxRepo.findOne({ aggregateId: code.id });
    expect(outboxEvent.eventType).toBe('CODE_UPDATED');
    expect(outboxEvent.status).toBe('PENDING');

    // 3. Outbox 폴러 실행
    await outboxPoller.processEvents();

    // 4. Kafka에 발행되었는지 확인
    const messages = await kafkaConsumer.consumeMessages('mdm.codes.events', { timeout: 5000 });
    expect(messages).toContainEqual(
      expect.objectContaining({
        key: code.id,
        value: expect.objectContaining({ eventType: 'CODE_UPDATED' })
      })
    );

    // 5. Outbox 상태 갱신 확인
    const updatedEvent = await outboxRepo.findOne({ id: outboxEvent.id });
    expect(updatedEvent.status).toBe('PUBLISHED');
  });
});
```

### E2E Test 시나리오 (Playwright)

```typescript
// ============================================================
// E2E 1: SSO Login -> Dashboard -> Project Access
// ============================================================
test.describe('SSO Login Flow', () => {
  test('SSO 로그인 후 대시보드 접근, 프로젝트 선택', async ({ page }) => {
    // 1. 로그인 페이지 이동
    await page.goto('/login');
    await expect(page.getByRole('heading', { name: '로그인' })).toBeVisible();

    // 2. SSO 로그인 버튼 클릭 → Keycloak 리디렉트
    await page.getByRole('button', { name: 'SSO 로그인' }).click();
    await expect(page).toHaveURL(/.*keycloak.*\/auth/);

    // 3. Keycloak에서 로그인
    await page.fill('#username', 'testuser');
    await page.fill('#password', 'TestPass1!');
    await page.click('#kc-login');

    // 4. 콜백 후 대시보드로 리디렉트
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('환영합니다')).toBeVisible();

    // 5. 프로젝트 목록 확인
    await expect(page.getByTestId('project-list')).toBeVisible();
    const projectCards = page.getByTestId('project-card');
    await expect(projectCards).toHaveCount(/* at least */ 1);

    // 6. 프로젝트 선택 → 프로젝트 대시보드
    await projectCards.first().click();
    await expect(page).toHaveURL(/\/projects\/.*\/dashboard/);
  });
});

// ============================================================
// E2E 2: Role Creation -> Permission Assignment -> Verification
// ============================================================
test.describe('Role & Permission Management', () => {
  test('역할 생성 → 권한 할당 → 사용자 배정 → 검증', async ({ page }) => {
    await loginAsAdmin(page);

    // 1. 역할 관리 페이지 이동
    await page.goto('/admin/roles');
    await page.getByRole('button', { name: '역할 추가' }).click();

    // 2. 역할 정보 입력
    await page.fill('[name="code"]', 'QA_LEAD');
    await page.fill('[name="name"]', 'QA 리더');
    await page.fill('[name="description"]', 'QA 팀 리더 역할');
    await page.selectOption('[name="scope"]', 'PROJECT');
    await page.getByRole('button', { name: '저장' }).click();

    // 3. 권한 할당
    await expect(page.getByText('역할이 생성되었습니다')).toBeVisible();
    await page.getByRole('tab', { name: '권한' }).click();
    await page.getByLabel('마스터 코드 조회').check();
    await page.getByLabel('마스터 코드 승인').check();
    await page.getByRole('button', { name: '권한 저장' }).click();

    // 4. 사용자 배정
    await page.goto('/projects/P1/members');
    await page.getByTestId('member-row-testuser').getByRole('button', { name: '역할 관리' }).click();
    await page.getByLabel('QA 리더').check();
    await page.getByRole('button', { name: '적용' }).click();

    // 5. 검증: 해당 사용자로 로그인 후 권한 확인
    await logout(page);
    await loginAs(page, 'testuser');
    await page.goto('/projects/P1/codes');
    await expect(page.getByRole('button', { name: '승인' })).toBeVisible();  // 승인 버튼 보임
  });
});

// ============================================================
// E2E 3: Master Code Creation -> Approval -> Query
// ============================================================
test.describe('Master Code Lifecycle', () => {
  test('코드 생성 요청 → 승인 → 조회 확인', async ({ page }) => {
    await loginAs(page, 'data-editor');

    // 1. 코드 생성 폼
    await page.goto('/projects/P1/codes/DEPT');
    await page.getByRole('button', { name: '코드 추가' }).click();

    await page.fill('[name="code"]', 'D999');
    await page.fill('[name="name_ko"]', '신규사업팀');
    await page.fill('[name="name_en"]', 'New Business Team');
    await page.getByRole('button', { name: '요청' }).click();

    // 2. 승인 대기 상태 확인
    await expect(page.getByText('승인 대기 중')).toBeVisible();

    // 3. 승인자 로그인
    await logout(page);
    await loginAs(page, 'data-owner');
    await page.goto('/approvals');

    // 4. 승인 처리
    const approvalRow = page.getByText('D999 - 신규사업팀');
    await approvalRow.click();
    await page.getByRole('button', { name: '승인' }).click();
    await page.fill('[name="comment"]', '확인 완료');
    await page.getByRole('button', { name: '확인' }).click();
    await expect(page.getByText('승인 처리되었습니다')).toBeVisible();

    // 5. 코드 조회 확인
    await page.goto('/projects/P1/codes/DEPT');
    await expect(page.getByText('D999')).toBeVisible();
    await expect(page.getByText('신규사업팀')).toBeVisible();
  });
});

// ============================================================
// E2E 4: Excel Bulk Upload -> Validation -> Application
// ============================================================
test.describe('Bulk Upload', () => {
  test('Excel 업로드 → 검증 → 승인 → 적용', async ({ page }) => {
    await loginAs(page, 'data-editor');

    // 1. 대량 업로드 페이지
    await page.goto('/projects/P1/codes/DEPT/import');

    // 2. 파일 업로드
    const fileInput = page.locator('input[type="file"]');
    await fileInput.setInputFiles('./fixtures/dept_codes_bulk.xlsx');

    // 3. 검증 결과 확인
    await expect(page.getByTestId('validation-summary')).toBeVisible({ timeout: 30000 });
    await expect(page.getByText('총 50건')).toBeVisible();
    await expect(page.getByText('성공: 48건')).toBeVisible();
    await expect(page.getByText('오류: 2건')).toBeVisible();

    // 4. 오류 행 확인
    await page.getByRole('tab', { name: '오류 목록' }).click();
    await expect(page.getByTestId('error-row')).toHaveCount(2);

    // 5. 유효한 건만 승인 요청
    await page.getByRole('button', { name: '유효 건 승인 요청' }).click();
    await expect(page.getByText('48건에 대한 승인 요청이 생성되었습니다')).toBeVisible();

    // 6. 승인자 처리
    await logout(page);
    await loginAs(page, 'data-owner');
    await page.goto('/approvals');
    await page.getByText('대량 업로드: DEPT').click();
    await page.getByRole('button', { name: '일괄 승인' }).click();
    await page.getByRole('button', { name: '확인' }).click();

    // 7. 적용 확인
    await expect(page.getByText('적용 완료')).toBeVisible();
    await page.goto('/projects/P1/codes/DEPT');
    // 기존 코드 + 48건이 추가되어 있음을 확인
  });
});
```

### 권한 시나리오 매트릭스

**Jakob Nielsen:** 다양한 역할과 리소스 조합에 대한 접근 권한을 매트릭스로 명확히 정의해야 합니다. 이것이 테스트의 기준선이 됩니다.

```
┌─────────────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ Role \ Action   │ 코드 조회│ 코드 생성│ 코드 수정│ 코드 삭제│ 코드 승인│ 대량 업로│
│                 │  (READ)  │ (CREATE) │ (UPDATE) │ (DELETE) │ (APPROVE)│ 드(IMPORT)│
├─────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ SYSTEM_ADMIN    │    ✓     │    ✓     │    ✓     │    ✓     │    ✓     │    ✓     │
│ PROJECT_ADMIN   │    ✓     │    ✓     │    ✓     │    ✓     │    ✓     │    ✓     │
│ DATA_OWNER      │    ✓     │    ✓     │    ✓     │    ✗     │    ✓     │    ✓     │
│ DATA_MANAGER    │    ✓     │    ✓     │    ✓     │    ✗     │    ✗     │    ✓     │
│ DATA_EDITOR     │    ✓     │    ✓     │    ✓     │    ✗     │    ✗     │    ✓     │
│ VIEWER          │    ✓     │    ✗     │    ✗     │    ✗     │    ✗     │    ✗     │
│ AUDITOR         │    ✓     │    ✗     │    ✗     │    ✗     │    ✗     │    ✗     │
│ GUEST           │    ✗     │    ✗     │    ✗     │    ✗     │    ✗     │    ✗     │
└─────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘

추가 시나리오:
┌─────────────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ Role \ Action   │ 역할 관리│ 사용자   │ 감사 로그│ 시스템   │ 프로젝트 │
│                 │          │ 관리     │ 조회     │ 설정     │ 설정     │
├─────────────────┼──────────┼──────────┼──────────┼──────────┼──────────┤
│ SYSTEM_ADMIN    │    ✓     │    ✓     │    ✓     │    ✓     │    ✓     │
│ PROJECT_ADMIN   │    ✓(P)  │    ✓(P)  │    ✓(P)  │    ✗     │    ✓     │
│ DATA_OWNER      │    ✗     │    ✗     │    ✓(P)  │    ✗     │    ✗     │
│ DATA_MANAGER    │    ✗     │    ✗     │    ✗     │    ✗     │    ✗     │
│ AUDITOR         │    ✗     │    ✗     │    ✓     │    ✗     │    ✗     │
│ (P) = 프로젝트 범위 내에서만                                          │
└─────────────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

### Performance Test (k6)

```javascript
// ============================================================
// k6 Performance Test Script
// ============================================================
import http from 'k6/http';
import { check, sleep, group } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const codeQueryTrend = new Trend('code_query_duration');

export const options = {
  stages: [
    { duration: '1m', target: 50 },    // Ramp-up
    { duration: '5m', target: 200 },   // Sustained load
    { duration: '2m', target: 500 },   // Peak load
    { duration: '1m', target: 0 },     // Ramp-down
  ],
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'],  // P95 < 200ms
    errors: ['rate<0.001'],                          // Error < 0.1%
    code_query_duration: ['p(95)<100'],              // 코드 조회 P95 < 100ms
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';
let authToken = '';

export function setup() {
  // 테스트용 토큰 발급
  const loginRes = http.post(`${BASE_URL}/api/v1/auth/login`, JSON.stringify({
    email: 'loadtest@test.com',
    password: 'LoadTest1!',
  }), { headers: { 'Content-Type': 'application/json' } });

  return { token: JSON.parse(loginRes.body).data.accessToken };
}

export default function (data) {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${data.token}`,
  };

  group('마스터 코드 조회', () => {
    const start = Date.now();
    const res = http.get(`${BASE_URL}/api/v1/codes/DEPT`, { headers });
    codeQueryTrend.add(Date.now() - start);

    check(res, {
      'status is 200': (r) => r.status === 200,
      'has data': (r) => JSON.parse(r.body).data.length > 0,
      'response time < 200ms': (r) => r.timings.duration < 200,
    });
    errorRate.add(res.status !== 200);
  });

  group('코드 상세 조회', () => {
    const res = http.get(`${BASE_URL}/api/v1/codes/DEPT/D001`, { headers });
    check(res, {
      'status is 200': (r) => r.status === 200,
      'has code field': (r) => JSON.parse(r.body).data.code === 'D001',
    });
    errorRate.add(res.status !== 200);
  });

  group('권한 체크 API', () => {
    const res = http.post(`${BASE_URL}/api/v1/auth/check-permission`,
      JSON.stringify({ resource: 'mdm:codes', action: 'READ' }),
      { headers }
    );
    check(res, {
      'status is 200': (r) => r.status === 200,
      'has allowed field': (r) => JSON.parse(r.body).data.allowed !== undefined,
    });
    errorRate.add(res.status !== 200);
  });

  sleep(1);
}

export function handleSummary(data) {
  return {
    'stdout': textSummary(data, { indent: ' ', enableColors: true }),
    './reports/k6-summary.json': JSON.stringify(data),
  };
}
```

**Kent Beck:** 훌륭합니다. 커버리지 목표와 구체적인 시나리오가 모두 정의되었습니다. 테스트 피라미드 구조가 균형 잡혀 있습니다.

**Andrew Ng:** k6 성능 테스트에서 코드 조회 P95 < 100ms 목표가 합리적입니다. 3계층 캐시 구조라면 충분히 달성 가능합니다.

**Donald Knuth:** 테스트 코드의 가독성도 중요합니다. 각 테스트 케이스의 이름이 한국어로 비즈니스 의도를 명확히 표현하고, Given-When-Then 구조를 따르고 있어 유지보수하기 좋습니다.

> **합의:** 테스트 전략 5단계 확정 — Unit(Vitest, 80%), Integration(Testcontainers, 70%), Contract(Pact, 핵심 100%), E2E(Playwright, Critical Path 100%), Performance(k6, P95<200ms). 권한 매트릭스 8개 역할 x 12개 액션 정의. E2E 4대 시나리오(SSO 로그인, 역할/권한 관리, 코드 생성/승인, 대량 업로드) 확정.

---

## TRD 핵심 결정 사항 요약

| # | 토의 | 결정 사항 | 내용 |
|---|------|----------|------|
| 1 | 7.1 아키텍처 | 아키텍처 스타일 | Micro Frontend + Backend 모듈러 모놀리스 (서비스 분리 준비) |
| 2 | 7.1 아키텍처 | Frontend 구조 | Module Federation 기반 Shell + Remote Apps |
| 3 | 7.1 아키텍처 | Backend 구조 | NestJS 모듈러 모놀리스, 도메인별 모듈 분리 |
| 4 | 7.1 아키텍처 | 통신 패턴 | 동기: REST API (서비스 간), 비동기: Kafka (이벤트 전파) |
| 5 | 7.1 아키텍처 | API Gateway | Kong Gateway — 인증, 라우팅, Rate Limiting |
| 6 | 7.1 아키텍처 | 인증/인가 | Keycloak(IdP) + JWT + OPA(정책 엔진) |
| 7 | 7.1 아키텍처 | 캐시 전략 | L1(메모리 5분) → L2(Redis 30분) → DB |
| 8 | 7.2 API 설계 | API 버전 관리 | URL 경로 방식 `/api/v1/` |
| 9 | 7.2 API 설계 | 응답 형식 | `{ success, data, error, meta }` 통일 Envelope |
| 10 | 7.2 API 설계 | Rate Limiting | 인증 API: 5req/min, 일반 API: 100req/min, 관리자: 500req/min |
| 11 | 7.2 API 설계 | 페이지네이션 | Cursor 기반 (대량 데이터), Offset 기반 (관리 화면) |
| 12 | 7.2 API 설계 | 에러 코드 체계 | `{SERVICE}_{DOMAIN}_{ERROR}` 형식 (e.g., `AUTH_TOKEN_EXPIRED`) |
| 13 | 7.3 프로젝트 구조 | 모노레포 도구 | Turborepo + pnpm workspace |
| 14 | 7.3 프로젝트 구조 | 디렉토리 컨벤션 | `apps/`, `packages/`, `services/`, `infra/` 4영역 |
| 15 | 7.3 프로젝트 구조 | TypeScript 설정 | strict: true, baseUrl 기반 경로 별칭 |
| 16 | 7.3 프로젝트 구조 | 공유 패키지 | `@pkg/ui`, `@pkg/sdk`, `@pkg/types`, `@pkg/utils` |
| 17 | 7.4 코딩 컨벤션 | 네이밍 규칙 | PascalCase(컴포넌트/클래스), camelCase(변수/함수), UPPER_SNAKE(상수) |
| 18 | 7.4 코딩 컨벤션 | 린팅/포매팅 | ESLint flat config + Prettier, husky pre-commit hook |
| 19 | 7.4 코딩 컨벤션 | 커밋 메시지 | Conventional Commits (`feat:`, `fix:`, `chore:` 등) |
| 20 | 7.5 DB 스키마 | DB 엔진 | PostgreSQL 16 (Auth DB + Master DB 논리 분리) |
| 21 | 7.5 DB 스키마 | Auth DB 테이블 수 | 15개 테이블 (users, roles, permissions 등) |
| 22 | 7.5 DB 스키마 | Master DB 테이블 수 | 10개 테이블 (code_groups, master_codes, outbox_events 등) |
| 23 | 7.5 DB 스키마 | 파티션 전략 | audit_logs: 월별 RANGE 파티션 (pg_partman 자동 생성) |
| 24 | 7.5 DB 스키마 | 인덱스 전략 | 조회 패턴 기반 복합 인덱스 + partial 인덱스 (outbox PENDING) |
| 25 | 7.5 DB 스키마 | 불변성 보장 | audit_logs UPDATE/DELETE 차단 트리거 |
| 26 | 7.5 DB 스키마 | 이벤트 발행 | Transactional Outbox 패턴 (outbox_events 테이블) |
| 27 | 7.6 시퀀스 | SSO 로그인 | OIDC Authorization Code Flow (Keycloak) |
| 28 | 7.6 시퀀스 | 권한 체크 | Gateway(인증) + OPA Sidecar(인가) 이중 구조 |
| 29 | 7.6 시퀀스 | 코드 조회 | SDK → L1 → L2 → API → DB 순차 캐시 탐색 |
| 30 | 7.6 시퀀스 | 코드 변경 승인 | 다단계 승인 + Outbox + Kafka 이벤트 발행 |
| 31 | 7.6 시퀀스 | MFE 로딩 | Registry API → 권한 체크 → CDN remoteEntry.js → mount() |
| 32 | 7.6 시퀀스 | Token 갱신 | 401 인터셉트 → Queue → 단일 refresh → retry |
| 33 | 7.7 환경 설정 | Node.js 버전 | 20.11.x LTS (.nvmrc 고정) |
| 34 | 7.7 환경 설정 | Frontend 핵심 | React 18.3, Vite 5.x, TailwindCSS 3.4, Zustand 4.x |
| 35 | 7.7 환경 설정 | Backend 핵심 | NestJS 10, TypeORM 0.3, KafkaJS 2.2 |
| 36 | 7.7 환경 설정 | 인프라 핵심 | Docker 25, K8s 1.29, Terraform 1.7 |
| 37 | 7.7 환경 설정 | 환경변수 관리 | `.env.example` Git 포함, `.env` Git 제외, 8개 그룹 분류 |
| 38 | 7.7 환경 설정 | 시크릿 관리 | HashiCorp Vault 1.15 (운영), .env (개발) |
| 39 | 7.7 환경 설정 | Docker Compose | 10개 서비스 (PG, Redis, Kafka, ES, Keycloak, Vault, Grafana 등) |
| 40 | 7.7 환경 설정 | 관측성 스택 | OpenTelemetry → Grafana + Loki + Tempo + Prometheus |
| 41 | 7.8 테스트 | Unit Test 도구 | Vitest — 커버리지 80% |
| 42 | 7.8 테스트 | Integration Test | Vitest + Testcontainers — 커버리지 70% |
| 43 | 7.8 테스트 | Contract Test | Pact 12 — Auth SDK, Master SDK 핵심 API 100% |
| 44 | 7.8 테스트 | E2E Test 도구 | Playwright 1.41 — Critical Path 100% |
| 45 | 7.8 테스트 | E2E 시나리오 | SSO 로그인, 역할/권한 관리, 코드 생성/승인, 대량 업로드 4개 |
| 46 | 7.8 테스트 | Performance Test | k6 0.48 — P95 < 200ms, Error < 0.1% |
| 47 | 7.8 테스트 | 권한 매트릭스 | 8개 역할 × 12개 액션 = 96개 조합 테스트 |
| 48 | 7.8 테스트 | API Mocking | MSW 2.2 (Mock Service Worker) |
