# Part 1: 시스템 아키텍처 브레인스토밍 종합 (Session 1~5)

## 토의 참석자

| # | 분야 | 인물 | 대표 철학 |
|---|------|------|-----------|
| 1 | AI | Andrew Ng + Andrej Karpathy | "AI is the new electricity" / "Software 2.0" |
| 2 | DB | Martin Kleppmann | "Data-Intensive Applications의 올바른 설계" |
| 3 | 아키텍처 | Martin Fowler | "Good programmers write code humans can understand." |
| 4 | SW | Robert C. Martin (Uncle Bob) | "Clean Code, SOLID Principles" |
| 5 | PM | Kent Beck (사회자) | "Make it work, make it right, make it fast" |
| 6 | UX/보안 | Jakob Nielsen + Bruce Schneier | "Usability is king" / "Security is a process, not a product" |
| 7 | 문서화 | Donald Knuth 스타일 | Literate Programming |

---

# 1. 전문가별 핵심 관점 및 검토 항목 (Session 1)

## 1-1. AI 관점 (Ng + Karpathy)

**핵심 관점:** 포털은 사용자 행동 데이터의 교차점이며, 마스터 데이터는 조직의 핵심 지식, 권한 시스템은 보안 이상 감지의 기반이다. 단, **AI 기능은 Phase 1에 넣으면 안 되며**, 데이터가 먼저 쌓여야 의미가 있다.

**AI가 즉시 가치를 만들 수 있는 3대 영역:**

1. **자연어 검색** - Embedding + Vector DB 기반, 마스터 데이터/프로젝트 리소스를 자연어로 검색
2. **이상 접근 탐지** - Isolation Forest/Autoencoder 기반, 접근 로그의 비정상 패턴 탐지
3. **PRD/TRD 초안 자동 생성** - LLM + RAG 기반, 기존 문서 패턴 학습

### AI 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **AI-1** | 자연어 검색 파이프라인 | Embedding 생성 파이프라인 설계. Vector DB(Milvus/Pinecone) 선택 및 인덱싱. 검색 결과 권한 필터링(Post/Pre-filtering). |
| **AI-2** | 이상 접근 탐지 모델 | 접근 로그 포맷 표준화(who/when/where/what/result). 베이스라인 학습 최소 3개월. 알림 채널 및 대응 워크플로우. |
| **AI-3** | LLM 통합 아키텍처 | 토큰 비용 관리. Rate Limiting 및 Fallback(GPT-4 → Claude → 로컬 모델). Prompt 버전 관리 및 A/B 테스트. |
| **AI-4** | 데이터 파이프라인 선행 설계 | Phase 1부터 수집 파이프라인 구축 필수. Event Sourcing 또는 CDC로 모든 변경 이력 보존. |
| **AI-5** | AI 추천 시스템 | 사용자별 개인화 추천. Cold Start 대응(역할 기반 기본 추천). |
| **AI-6** | AI 윤리 및 거버넌스 | 설명 가능성(Explainability). 편향 모니터링. 사용자 동의 및 투명성. |

---

## 1-2. DB 관점 (Kleppmann)

**핵심 관점:** 데이터 모델이 애플리케이션을 지배한다. 각 프로젝트의 이질적 데이터 모델(같은 "부서" 개념이 컬럼명/타입/계층 표현이 전부 다름)을 통합하려면 **Canonical Data Model(표준 데이터 모델)**이 반드시 선행되어야 한다.

### DB 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **DB-1** | 마스터 데이터 표준 모델 | 공통 코드/조직/사용자 Canonical Schema 정의. 계층형 vs 플랫 vs 혼합 코드 체계. SCD Type 2 적용 여부. |
| **DB-2** | 데이터 동기화 전략 | MDM Hub 동기화: CDC(Debezium) vs API Polling vs Event-Driven. 충돌 해결: Master Wins / Last-Write-Wins / Manual Merge. Eventual Consistency 허용 범위 정의. |
| **DB-3** | 권한 데이터 모델 | RBAC 기본 엔티티(User/Role/Permission/Resource). ProjectMembership 스코핑. 리소스 계층(Org > Project > Module > Feature > Data Row). 위임 모델. |
| **DB-4** | 감사 로그 저장소 | 불변 저장소(Append-only PostgreSQL/OpenSearch/ClickHouse). 보존 정책: Hot(3개월) / Warm(1년) / Cold(5년). 로그 스키마(who/when/where/what/how/result). |

---

## 1-3. 아키텍처 관점 (Fowler)

**핵심 원칙:** "Integration at the edge, autonomy at the core" - 각 프로젝트는 내부적으로 자율적이고, 통합은 3개 경계면에서만 발생한다.

**3대 통합 경계면:**

| 경계면 | 대상 | 통합 지점 |
|--------|------|-----------|
| Frontend 통합 | 사용자가 하나의 포털에서 여러 프로젝트 접근 | Shell App (레이아웃, 인증, 네비게이션) |
| 인증/인가 통합 | 모든 프로젝트가 동일한 인증 시스템 사용 | Auth Service + Policy Engine |
| 데이터 통합 | 공통 데이터를 중앙 관리 | MDM Hub + API Gateway |

> 이 3개 경계면 외에 프로젝트 간 직접 의존을 만들면 시스템이 스파게티가 된다.

### 아키텍처 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **ARCH-1** | Micro Frontend 아키텍처 | Shell App 책임 범위. Remote App 계약(bootstrap/mount/unmount). 통합 방식: Module Federation vs Single-SPA vs iframe. 공유 디펜던시 전략. |
| **ARCH-2** | 인증/인가 아키텍처 | SSO(OIDC + PKCE). Token 전략: Access(JWT, 15분) + Refresh(Opaque, 7일). 인가: API Gateway / Service / 양쪽. 세션 관리. |
| **ARCH-3** | API 아키텍처 | API Gateway 역할(라우팅/Rate Limiting/인증/로깅). 내부 API 표준(REST OpenAPI 3.1 또는 gRPC). 버전 관리. BFF 패턴 여부. |
| **ARCH-4** | 장애 격리 및 복원 | Circuit Breaker, Bulkhead, Timeout/Retry, Graceful Degradation. |

---

## 1-4. SW 관점 (Uncle Bob)

**핵심 관점:** 시스템 성패는 **Contract-First 설계**에 달려 있다. 계약 없이 구현부터 시작하면 2개월 뒤 통합 불가능.

**3대 제안:**

1. **Contract-First** - OpenAPI 스펙을 먼저 쓰고, 거기서 클라이언트/서버 코드 생성
2. **SDK 제공** - Auth, Master Data, Event Bus 공통 SDK → 프로젝트별 중복 구현 방지
3. **자동화 테스트 3대 원칙** - Contract Test(Pact), Integration Test(Playwright E2E), Chaos Test(장애 시 생존 검증)

### SW 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **SW-1** | Contract-First 개발 프로세스 | OpenAPI 3.1 중앙 저장소. 코드 자동 생성(openapi-generator). Breaking Change 감지. Semantic Versioning + Deprecation 정책. |
| **SW-2** | 공통 SDK 설계 | `@portal/auth-sdk`, `@portal/master-sdk`, `@portal/event-sdk`, `@portal/ui-sdk`. npm private registry 배포. |
| **SW-3** | 테스트 전략 | Unit(커버리지 80%+), Contract(Pact), Integration(Playwright E2E), Performance(k6/Artillery). |
| **SW-4** | 코드 품질 가드레일 | ESLint/Prettier 통일. Husky + lint-staged. SonarQube(중복률 3% 이하, 취약점 0). Dependency Bot. |

---

## 1-5. PM 관점 (Beck)

**핵심 관점:** **Tracer Bullet** 접근법 - 플랫폼을 먼저 만들지 말고, 하나의 대표적 프로젝트를 끝에서 끝까지 얇게 관통하는 동작하는 시스템을 먼저 만든다.

> **"플랫폼을 먼저 만들지 말고, 프로젝트를 올리면서 플랫폼을 함께 키우세요."**

Tracer Bullet 시나리오 예시: SSO 로그인 → 포털 메인 진입 → 프로젝트 A 대시보드 → 공통 코드로 데이터 표시 → 로그아웃

- 대상은 "가장 단순한" 프로젝트가 아니라 **"가장 대표적인" 프로젝트**를 선택해야 일반적인 문제를 발견할 수 있다.

### PM 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **PM-1** | Tracer Bullet 대상 선정 | 선정 기준: 대표성, 팀 역량, 일정 여유. E2E 시나리오 정의. |
| **PM-2** | 단계별 롤아웃 계획 | Phase 0~4 로드맵 (상세는 3-3절 참조). |
| **PM-3** | 이해관계자 관리 | 통합 부담 최소화(SDK + 가이드 + 페어 프로그래밍). 독립 배포 능력 유지 보장. Go/No-Go 기준 정의. |
| **PM-4** | 위험 관리 | 팀 저항 → SDK 품질로 극복. 기술 스택 불일치 → Adapter 패턴. 데이터 품질 → 거버넌스 위원회. 성능 저하 → 성능 버짓 + 모니터링. |

---

## 1-6. UX 관점 (Nielsen)

**핵심 관점:** 포털은 사용자의 **매일의 업무 진입점(Daily Entry Point)**이므로, UX는 "예쁜 화면"이 아니라 **업무 효율성**의 핵심이다.

**2대 강조점:**

1. **Design Token 필수** - 6개 프로젝트의 시각적 일관성 확보, 인지 부하 감소
2. **통합 검색이 포털의 생명줄** - 6개 프로젝트에 걸친 데이터를 하나의 검색창에서 검색

### UX 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **UX-1** | Design Token 표준 | 색상/타이포그래피/간격/그림자/모서리 반경 변수 정의. 다크 모드(CSS 변수 기반). 프레임워크별 변환. 접근성 WCAG 2.1 AA 이상. |
| **UX-2** | 통합 검색 UX | 검색 범위: 마스터 데이터/프로젝트 리소스/메뉴/사용자/문서. 카테고리별 그룹핑 + 권한 필터링. 개인화(최근 검색/자주 사용). Cmd/Ctrl+K 단축키. |

---

## 1-7. 보안 관점 (Schneier)

**핵심 관점:** 보안은 제품이 아니라 프로세스.

**2대 원칙:**

1. **Zero Trust** - 내부 API 호출이라도 인증 토큰 필수. 모든 요청 인증+인가, 최소 권한, 지속적 검증, 모든 접근 기록.
2. **Audit Trail 의무** - 누가/언제/어떤 데이터에 접근·변경했는지 전부 기록 (보안 사고 추적 + 규정 준수).

### 보안 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **SEC-1** | Zero Trust 구현 | mTLS 또는 Service Mesh 기반 서비스 간 인증. API Gateway JWT 검증 + Rate Limiting. Service Account Token 관리. 프로젝트별 네트워크 격리. |
| **SEC-2** | 감사 로그 체계 | 모든 인증/인가/데이터 접근 이벤트 수집. 변조 방지(Hash Chain/WORM). 실시간 이상 탐지 + 주기적 감사 리포트. 보존 기간 및 접근 제한 규정 준수. |
| **SEC-3** | 취약점 관리 프로세스 | OWASP Top 10 기반 SAST/DAST. 의존성 취약점 스캔(Snyk/Trivy). 보안 사고 대응 계획(탐지→격리→분석→복구→사후 검토). 분기 1회 침투 테스트. |

---

# 2. 핵심 쟁점 교차 토론 및 합의 (Session 2)

## 쟁점 1: 권한 모델 - RBAC vs ABAC

**결론: RBAC + Project Scope + Policy Engine(OPA)**

- ABAC는 이상적이나 정책 작성·관리 역량 부재로 현실적이지 않음 (Schneier, Uncle Bob)
- 순수 RBAC는 역할 폭발 위험 (Fowler)
- **해결책:** 역할 5개로 제한 + 프로젝트 스코프로 적용 (Ng)

```
역할: Super Admin, Project Admin, Editor, Viewer, Guest

데이터 모델:
  ProjectMembership {
    user_id, project_id, role, granted_at, granted_by, expires_at
  }
```

- 세밀한 정책이 필요한 경우(10%) OPA로 확장 (Fowler)

```
요청 → [1] RBAC 체크 (빠름, 캐시 가능)
         ├── 명확한 허용/거부 → 완료
         └── 추가 정책 필요 → [2] OPA 정책 엔진 → 허용/거부
```

- 모든 권한 변경은 감사 로그에 기록 (Schneier)
- Phase 3에서 AI 기반 이상 패턴 탐지 추가 (Karpathy)

> **합의:** RBAC + Project Scope 기본, OPA 확장. 역할 5개 제한. 모든 권한 변경 감사 로그 기록.

---

## 쟁점 2: 마스터 데이터 통합 전략

**결론: MDM Hub(Golden Source) + REST API + Event Bus + 로컬 캐시**

| 선택지 | 장점 | 단점 |
|--------|------|------|
| 중앙 집중형 MDM | 데이터 일관성 보장 | SPOF, 자율성 제한 |
| 분산형 (Data Mesh) | 자율성, 장애 격리 | 불일치, 거버넌스 어려움 |
| **하이브리드 (채택)** | **일관성 + 가용성** | **구현 복잡도** |

**하이브리드 구조:**

```
MDM Hub (Golden Source)
    ├── REST API: 실시간 조회 (캐시 Redis TTL 5분)
    ├── Event Bus: 변경 알림 (Kafka/RabbitMQ) → 로컬 캐시 갱신
    └── Bulk Sync: 일 1회 배치 (정합성 검증)
```

**장애 대응:** SDK에 Circuit Breaker 내장 → MDM Hub 장애 시 로컬 캐시 fallback

**거버넌스 체계:**
- Data Steward: 도메인별 담당자 (코드/조직/사용자)
- Data Quality Rule: 필수 필드, 형식, 참조 무결성, 중복 탐지
- 변경 승인 워크플로우: 요청 → 자동 검증 → Data Steward 승인 → 적용 → 알림

> **합의:** MDM Hub + API + Event Bus + 로컬 캐시. SDK에 Circuit Breaker 내장. 거버넌스 체계 필수. Phase 3에서 AI 품질 검증 추가.

---

## 쟁점 3: 다중 기술 스택 포털 통합

**결론: Micro Frontend(Module Federation) + 프레임워크 독립적 Contract**

**Shell-Remote 계약 정의:**

```typescript
interface RemoteAppContract {
  bootstrap(): Promise<void>;
  mount(container: HTMLElement, context: ShellContext): Promise<void>;
  unmount(): Promise<void>;
  manifest: AppManifest;
}

interface AppManifest {
  id: string;           // "project-a"
  name: string;         // "프로젝트 A"
  version: string;      // "1.2.0"
  basePath: string;     // "/project-a"
  entry: string;        // remoteEntry.js URL
  framework: 'react' | 'vue' | 'angular' | 'other';
  routes: RouteDefinition[];
  requiredPermissions: string[];
  loadStrategy: 'module-federation' | 'single-spa' | 'iframe';
}

interface ShellContext {
  auth: AuthContext;
  navigation: NavigationContext;
  events: EventBusContext;
}
```

**프레임워크별 Adapter 제공:** `@portal/react-adapter`, `@portal/vue-adapter`, `@portal/angular-adapter`

**공유 디펜던시 전략:**

| 공유 대상 (singleton) | 공유하지 않는 것 |
|----------------------|-----------------|
| react, react-dom | 프로젝트별 비즈니스 로직 |
| @portal/auth-sdk | 프로젝트별 상태 관리 (Redux 등) |
| @portal/master-sdk, event-sdk | 프로젝트별 API 클라이언트 |
| @portal/design-tokens | |

**통합 전략 요약:**

1. Contract: `@portal/shell-contracts` (TypeScript 인터페이스)
2. Adapter: 프레임워크별 어댑터 패키지
3. Design: `@portal/design-tokens` (CSS 변수, SCSS, Tailwind Config)
4. SDK: `@portal/auth-sdk`, `@portal/master-sdk`, `@portal/event-sdk`

> **합의:** Module Federation 기본 + 프레임워크 독립적 Contract + Adapter + Design Token + 공유 SDK. 공유 디펜던시는 Shell 관련 SDK만.

---

# 3. PRD/TRD 체크리스트 및 로드맵 (Session 3)

## 3-1. PRD 필수 검토 항목

| # | PRD 항목 | 설명 | 담당 |
|---|----------|------|------|
| P-1 | 프로젝트 개요 및 비전 | 다중 프로젝트 포털의 존재 이유와 비즈니스 가치 | PM |
| P-2 | 사용자 페르소나 | 최소 3종: 일반 사용자, 프로젝트 관리자, 시스템 관리자 | UX |
| P-3 | 핵심 사용자 시나리오 | 로그인→포털→프로젝트 접근→마스터 데이터 활용→로그아웃 E2E | UX/PM |
| P-4 | 기능 요구사항 (포털) | Shell App, Remote App 통합, 통합 검색, 알림, 메뉴 관리 | 아키텍처 |
| P-5 | 기능 요구사항 (권한 관리) | RBAC + Project Scope, 역할 관리, 위임, 감사 로그 | 보안/DB |
| P-6 | 기능 요구사항 (마스터 데이터) | 코드/조직/사용자 관리, 변경 승인, 엑셀 연동 | DB/UX |
| P-7 | 비기능 요구사항 | 성능, 가용성(99.9%), 보안 등급, 접근성(WCAG AA) | 전체 |
| P-8 | 통합 요구사항 | 기존 프로젝트 통합, SSO 연동, 외부 시스템 인터페이스 | 아키텍처 |
| P-9 | 성공 지표(KPI) | 포털 활성 사용률, 평균 앱 전환 시간, 마스터 데이터 정합성 | PM |
| P-10 | 제약 조건 및 가정 | 기존 시스템/기술 스택/조직/일정 제약 | PM |

## 3-2. TRD 필수 검토 항목

| # | TRD 항목 | 설명 | 담당 |
|---|----------|------|------|
| T-1 | 시스템 아키텍처 다이어그램 | 전체 컴포넌트 배치도, 통신 흐름, 기술 스택 | 아키텍처 |
| T-2 | Micro Frontend 설계 | Shell App 구조, Remote App 계약, Module Federation, 라우팅 | 아키텍처/SW |
| T-3 | 인증/인가 설계 | SSO(OIDC+PKCE) 흐름, JWT 구조, RBAC 모델, OPA 정책 예시 | 보안/DB |
| T-4 | 마스터 데이터 설계 | Canonical Schema, API 명세, 동기화 전략, Circuit Breaker | DB |
| T-5 | API 설계 | OpenAPI 3.1 스펙, 엔드포인트 목록, 입출력 스키마, 에러 코드 | SW |
| T-6 | 데이터베이스 설계 | ERD, 테이블 정의서, 인덱스 전략, 파티셔닝 | DB |
| T-7 | SDK 설계 | 각 SDK의 API, 내부 구조, 의존성, 사용 예시 | SW |
| T-8 | Design System 설계 | Design Token 정의, 컴포넌트 목록, Storybook 구조 | UX |
| T-9 | 인프라/배포 설계 | 컨테이너 구조, CI/CD, 환경별 설정, 모니터링 | 아키텍처 |
| T-10 | 보안 설계 | Zero Trust, 감사 로그, 취약점 관리, 암호화 전략 | 보안 |
| T-11 | 테스트 전략 | Unit/Contract/Integration/E2E/Performance 테스트 계획 | SW |
| T-12 | 디렉토리 구조 | 모노레포 구조, 패키지 구성, 파일 명명 규칙 | SW |
| T-13 | 환경 변수 및 설정 | 환경별 변수, Feature Flags, 외부 서비스 연결 정보 | 아키텍처/SW |

## 3-3. 우선순위 로드맵

| Phase | 이름 | 기간 | 핵심 목표 | 주요 산출물 |
|-------|------|------|-----------|-------------|
| **0** | 설계 및 기반 | 4주 | Contract 정의, 인프라 셋업, SDK 스켈레톤 | OpenAPI 스펙, Shell App 스켈레톤, CI/CD, Design Token v1, SDK 인터페이스 |
| **1** | Tracer Bullet | 6주 | 첫 번째 프로젝트 E2E 통합 | SSO 동작, Shell + 1 Remote App, 기본 RBAC, 마스터 데이터 API, 기본 감사 로그 |
| **2** | 확장 | 8주 | 2~3개 프로젝트 추가 | 추가 Remote App, 마스터 데이터 관리 화면, 권한 관리 화면, 통합 검색 v1, 알림 |
| **3** | 고도화 | 8주 | AI 기능, 고급 권한, 모니터링 | 자연어 검색, 이상 접근 탐지 v1, OPA 통합, 감사 대시보드 |
| **4** | 최적화 | 지속 | 개인화, 자동화, 운영 성숙 | AI 추천, PRD/TRD 자동 초안, A/B 테스트, 셀프서비스 온보딩 |

## 3-4. Phase 간 Go/No-Go 기준

**Phase 0 → 1:**
- [ ] OpenAPI 스펙 최소 3개 서비스 완료
- [ ] Shell App 스켈레톤 로컬 구동
- [ ] CI/CD로 스테이징 배포 가능
- [ ] Design Token Storybook 확인 가능

**Phase 1 → 2:**
- [ ] Tracer Bullet 시나리오 E2E 성공
- [ ] 첫 번째 Remote App 포털 정상 동작
- [ ] SSO 로그인/로그아웃 정상
- [ ] 마스터 데이터 API 응답 200ms 이하
- [ ] 기본 RBAC 권한 체크 정상

**Phase 2 → 3:**
- [ ] 최소 3개 프로젝트 포털 통합 완료
- [ ] 마스터 데이터 관리 화면 CRUD 가능
- [ ] 권한 관리 화면 역할 부여/회수 가능
- [ ] 통합 검색 프로젝트 간 결과 표시
- [ ] E2E 테스트 통과율 95% 이상

**Phase 3 → 4:**
- [ ] AI 검색 정확도(MRR@10) 0.7 이상
- [ ] 이상 접근 탐지 False Positive 10% 이하
- [ ] 감사 대시보드 30일 내 접근 조회 가능
- [ ] P95 응답 시간 500ms 이하

---

# 4. 보완 영역 (Session 4)

## 4-1. DevOps / Observability

**Observability 3대 기둥:**

| 기둥 | 도구 | 핵심 |
|------|------|------|
| Logs | Fluentd/Vector → OpenSearch/Loki | JSON 형식, 필수 필드(timestamp/service/level/trace_id/user_id/message) |
| Metrics | Prometheus + Grafana | RED 메트릭(Rate/Errors/Duration), 서비스별 + 엔드포인트별 |
| Traces | OpenTelemetry → Jaeger/Tempo | W3C Trace Context 헤더로 전체 호출 체인 추적 |

| ID | 항목 | 상세 |
|----|------|------|
| **OPS-1** | CI/CD 파이프라인 | 모노레포(Turborepo/Nx), Blue-Green/Canary 배포, Shell/Remote 독립 배포, dev→staging→production 3단계. |
| **OPS-2** | 구조화 로그 표준 | JSON 스키마 표준화. 로그 레벨 정책(ERROR: 즉시 알림, WARN: 대시보드, INFO: 분석). 민감 정보 마스킹. |
| **OPS-3** | 분산 추적 | OpenTelemetry SDK 전체 서비스 내장. Frontend에서도 Trace 시작. 샘플링: Head-based 10% + Error 100%. |
| **OPS-4** | 알림 및 온콜 | Slack/Teams + PagerDuty/OpsGenie. P1~P4 등급. Runbook. SLO/SLI(가용성 99.9%, P95 500ms, 에러율 0.1%). |

## 4-2. 개발자 경험 (DX)

**이상적인 프로젝트 온보딩 (3단계):**

1. SDK 설치: `npm install @portal/react-adapter @portal/auth-sdk @portal/master-sdk`
2. Adapter로 앱 감싸기: `export default createReactRemoteApp(MyApp, myManifest);`
3. 포털 등록: `portal-cli register --manifest ./manifest.json`

| ID | 항목 | 상세 |
|----|------|------|
| **DX-1** | CLI 도구 (portal-cli) | 스캐폴딩(`create`), 로컬 개발(`dev`), 포털 등록(`register`), 배포(`deploy`). |
| **DX-2** | 개발자 문서 포털 | Swagger UI(API), SDK 가이드, Storybook(Design System), 트러블슈팅 FAQ. |
| **DX-3** | 로컬 개발 환경 | Shell App 없이 독립 개발/테스트. Mock Auth Context. Mock Master Data. Hot Reload. |

## 4-3. 데이터 거버넌스 심화

**데이터 분류 등급:**

| Level | 등급 | 예시 | 요구사항 |
|-------|------|------|----------|
| 1 | 공개 | 시스템 코드명, 메뉴 이름 | 기본 접근 제어 |
| 2 | 내부 | 조직 구조, 프로젝트 목록 | 인증된 사용자만 |
| 3 | 기밀 | 개인 연락처, 급여 정보 | 암호화 저장 + 감사 로그 |
| 4 | 극비 | 인사 평가, 보안 감사 결과 | 암호화 + 접근 승인 필수 |

| ID | 항목 | 상세 |
|----|------|------|
| **GOV-1** | 데이터 리니지 추적 | 원천 매핑, 생성→전파→소비 경로, 변경 영향도 분석, 변경 전 영향도 리포트. |
| **GOV-2** | 데이터 분류 및 보호 | 4단계 분류, 필드 레벨 태깅, 등급별 접근/암호화/마스킹/감사 정책, 분기 1회 검토. |
| **GOV-3** | 데이터 품질 관리 | 품질 지표(완전성/정확성/일관성/적시성), 자동 검사, 품질 대시보드, 이상 데이터 알림. |

## 4-4. 성능(Performance)

**성능 기준 (Nielsen 기반):**

| 항목 | 목표 |
|------|------|
| 포털 초기 로딩 (FCP) | 3초 이내 |
| 앱 전환 | 1초 이내 (프리페칭 시 0.5초) |
| API 응답 (P95) | 200ms 이내 |
| 마스터 데이터 조회 (캐시 히트) | 100ms 이내 |
| 검색 결과 표시 | 500ms 이내 |

**Performance Budget:**

| 구분 | 항목 | 상한 |
|------|------|------|
| Frontend | Remote App 초기 번들 (gzip) | 200KB |
| Backend | API P95 / P99 | 200ms / 500ms |
| Backend | DB 쿼리 | 50ms |
| Backend | 외부 서비스 Circuit Breaker 타임아웃 | 3초 |

| ID | 항목 | 상세 |
|----|------|------|
| **PERF-1** | Performance Budget 정의 | Frontend/Backend 상한 설정. CI에서 자동 체크, 초과 시 PR 차단. |
| **PERF-2** | 캐싱 전략 | CDN(정적 리소스), Redis(마스터 데이터 TTL 5분, 권한 TTL 1분), Service Worker, Event Bus 무효화. |
| **PERF-3** | 부하 테스트 계획 | k6/Artillery, 동시 1000명, P95 500ms 이내/에러율 0.1%. 주 1회 + 릴리즈 전 필수. |

## 4-5. 마이그레이션 전략

**Strangler Fig Pattern 적용:**

| 단계 | 전략 |
|------|------|
| Phase 1: 공존 | 기존 시스템과 포털 동시 운영, 양쪽 모두 접근 가능 |
| Phase 2: 전환 | 새 기능은 포털에서만, 사용 빈도 낮은 기능부터 이전 |
| Phase 3: 완료 | 기존 시스템 → 포털 리다이렉트 → READ-ONLY → 폐기 |

**프로젝트 마이그레이션 체크리스트 (10단계):**

1. SDK 의존성 추가
2. 인증 로직 → `@portal/auth-sdk`로 교체
3. 마스터 데이터 조회 → `@portal/master-sdk`로 교체
4. Module Federation 설정 추가
5. Remote Entry 파일 작성
6. manifest.json 작성
7. Design Token 적용
8. 독립 실행 모드 유지 (포털 없이도 단독 실행 가능)
9. Contract Test 작성
10. 스테이징 통합 테스트

| ID | 항목 | 상세 |
|----|------|------|
| **MIG-1** | Strangler Fig 전략 | 공존 기간 최소 3개월. 트래픽 라우팅(URL/Feature Flag). 롤백 계획. 데이터 동기화. |
| **MIG-2** | 프로젝트별 마이그레이션 가이드 | 10단계 체크리스트 + 프레임워크별 예시. 페어 프로그래밍 지원. |
| **MIG-3** | 독립 실행 모드 유지 | Dual-mode: 포털 내부 + 단독 실행. 포털 장애 시에도 독립 서비스 가능. |

## 4-6. 다국어(i18n) / 멀티 테넌시

**i18n 아키텍처:**
- Shell App: `@portal/i18n` (공통), Remote App: 각자 i18n, 마스터 데이터: DB 다국어 컬럼
- Shell에서 로케일 설정 → `AppContext.i18n.locale`로 Remote App에 전달
- 날짜/숫자: `Intl` API 기반

**멀티 테넌시:** 현재 Level 1 준비(tenant_id 컬럼 예약, 단일 값), 필요 시 Row-Level Security 활성화

| ID | 항목 | 상세 |
|----|------|------|
| **EXT-1** | i18n 아키텍처 | Shell→Remote 로케일 전달. 메시지 키 규칙(`<domain>.<module>.<key>`). Intl API 포맷. |
| **EXT-2** | 멀티 테넌시 준비 | `tenant_id` 컬럼 예약. `X-Tenant-ID` 헤더 규약. Feature Flags/Design Token 오버라이드. |
| **EXT-3** | 확장성 설계 원칙 | Stateless 서비스(세션 Redis). 데이터 파티셔닝. Auto-scaling(CPU/메모리 + 커스텀 메트릭). |

---

# 5. 전문가 클로징 메시지 (Session 5)

| 전문가 | 핵심 메시지 |
|--------|-------------|
| **Andrew Ng** | Phase 1부터 데이터 파이프라인을 깔아두세요. AI는 데이터가 쌓인 후에 빛을 발합니다. |
| **Andrej Karpathy** | 자연어 검색, 이상 탐지, 문서 자동 생성이 이 포털을 차별화할 것입니다. |
| **Martin Kleppmann** | Canonical Data Model 없이 마스터 데이터 통합은 환상입니다. 거버넌스가 기술보다 먼저입니다. |
| **Martin Fowler** | Integration at the edge, autonomy at the core. 3개 경계면 외에는 결합하지 마세요. |
| **Uncle Bob** | Contract-First, SDK 제공, 자동화 테스트. 이 세 가지 없이 6개 프로젝트 통합은 카오스입니다. |
| **Kent Beck** | Tracer Bullet으로 시작하세요. 플랫폼을 먼저 만들지 말고, 프로젝트를 올리면서 키우세요. |
| **Jakob Nielsen** | Design Token과 통합 검색이 매일의 업무 진입점인 이 포털의 UX 핵심입니다. |
| **Bruce Schneier** | Zero Trust와 Audit Trail은 선택이 아닙니다. 모든 요청을 검증하고, 모든 접근을 기록하세요. |

---

# 6. 전체 46개 검토 항목 요약

| # | ID | 영역 | 항목 | Phase |
|---|-----|------|------|-------|
| 1 | AI-1 | AI | 자연어 검색 파이프라인 | 3 |
| 2 | AI-2 | AI | 이상 접근 탐지 모델 | 3 |
| 3 | AI-3 | AI | LLM 통합 아키텍처 | 3 |
| 4 | AI-4 | AI | 데이터 파이프라인 선행 설계 | 1 |
| 5 | AI-5 | AI | AI 추천 시스템 | 4 |
| 6 | AI-6 | AI | AI 윤리 및 거버넌스 | 3 |
| 7 | DB-1 | DB | 마스터 데이터 표준 모델 | 0 |
| 8 | DB-2 | DB | 데이터 동기화 전략 | 1 |
| 9 | DB-3 | DB | 권한 데이터 모델 | 0 |
| 10 | DB-4 | DB | 감사 로그 저장소 | 1 |
| 11 | ARCH-1 | 아키텍처 | Micro Frontend 아키텍처 | 0 |
| 12 | ARCH-2 | 아키텍처 | 인증/인가 아키텍처 | 0 |
| 13 | ARCH-3 | 아키텍처 | API 아키텍처 | 0 |
| 14 | ARCH-4 | 아키텍처 | 장애 격리 및 복원 | 1 |
| 15 | SW-1 | SW | Contract-First 개발 프로세스 | 0 |
| 16 | SW-2 | SW | 공통 SDK 설계 | 0 |
| 17 | SW-3 | SW | 테스트 전략 | 1 |
| 18 | SW-4 | SW | 코드 품질 가드레일 | 0 |
| 19 | PM-1 | PM | Tracer Bullet 대상 선정 | 0 |
| 20 | PM-2 | PM | 단계별 롤아웃 계획 | 0 |
| 21 | PM-3 | PM | 이해관계자 관리 | 0 |
| 22 | PM-4 | PM | 위험 관리 | 0 |
| 23 | UX-1 | UX | Design Token 표준 | 0 |
| 24 | UX-2 | UX | 통합 검색 UX | 2 |
| 25 | SEC-1 | 보안 | Zero Trust 구현 | 1 |
| 26 | SEC-2 | 보안 | 감사 로그 체계 | 1 |
| 27 | SEC-3 | 보안 | 취약점 관리 프로세스 | 1 |
| 28 | OPS-1 | DevOps | CI/CD 파이프라인 설계 | 0 |
| 29 | OPS-2 | DevOps | 구조화 로그 표준 | 1 |
| 30 | OPS-3 | DevOps | 분산 추적 | 1 |
| 31 | OPS-4 | DevOps | 알림 및 온콜 체계 | 1 |
| 32 | DX-1 | DX | CLI 도구 (portal-cli) | 1 |
| 33 | DX-2 | DX | 개발자 문서 포털 | 1 |
| 34 | DX-3 | DX | 로컬 개발 환경 | 0 |
| 35 | GOV-1 | 거버넌스 | 데이터 리니지 추적 | 2 |
| 36 | GOV-2 | 거버넌스 | 데이터 분류 및 보호 | 1 |
| 37 | GOV-3 | 거버넌스 | 데이터 품질 관리 | 2 |
| 38 | PERF-1 | 성능 | Performance Budget 정의 | 0 |
| 39 | PERF-2 | 성능 | 캐싱 전략 | 1 |
| 40 | PERF-3 | 성능 | 부하 테스트 계획 | 2 |
| 41 | MIG-1 | 마이그레이션 | Strangler Fig 전략 | 1 |
| 42 | MIG-2 | 마이그레이션 | 프로젝트별 마이그레이션 가이드 | 1 |
| 43 | MIG-3 | 마이그레이션 | 독립 실행 모드 유지 | 1 |
| 44 | EXT-1 | 확장 | i18n 아키텍처 | 1 |
| 45 | EXT-2 | 확장 | 멀티 테넌시 준비 | 2 |
| 46 | EXT-3 | 확장 | 확장성 설계 원칙 | 1 |

---

> **다음 단계:** Part 2(포털 설계 심화 토의)에서 Micro Frontend 통합 방식, Shell App 설계, Design System 전략을 구체화합니다.
