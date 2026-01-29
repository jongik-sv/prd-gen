# Part 1: 시스템 아키텍처 브레인스토밍 - 대화형 토론 기록 (Session 1~5)

## 토의 참석자

| # | 분야 | 인물 | 대표 철학 |
|---|------|------|-----------|
| 1 | AI | Andrew Ng + Andrej Karpathy | "AI is the new electricity" / "Software 2.0" |
| 2 | DB | Martin Kleppmann | "Data-Intensive Applications의 올바른 설계" |
| 3 | 아키텍처 | Martin Fowler | "Any fool can write code a computer can understand. Good programmers write code humans can understand." |
| 4 | SW | Robert C. Martin (Uncle Bob) | "Clean Code, SOLID Principles" |
| 5 | PM | Kent Beck (사회자) | "Make it work, make it right, make it fast" |
| 6 | UX/보안 | Jakob Nielsen + Bruce Schneier | "Usability is king" / "Security is a process, not a product" |
| 7 | 문서화 | Donald Knuth 스타일 | Literate Programming |

---

# Session 1: 오프닝 - 각 전문가의 핵심 관점

**Beck (사회자):** 오늘부터 다중 프로젝트 포털, 권한 관리, 마스터 데이터 관리 시스템의 PRD/TRD 설계를 위한 브레인스토밍을 시작합니다. 각 전문가가 자신의 관점에서 이 시스템을 바라보는 **핵심 시각**을 먼저 공유해주세요. Ng부터 시작하시죠.

---

## 1-1. AI 관점 (Andrew Ng + Andrej Karpathy)

**Ng:** 감사합니다. 이 시스템은 단순한 포털이 아닙니다. 다양한 프로젝트의 데이터가 흐르는 **교차점**이고, 그 교차점에서 AI가 가치를 만들 수 있습니다. 포털은 사용자의 행동 데이터가 가장 풍부하게 모이는 곳이고, 마스터 데이터는 조직의 핵심 지식이며, 권한 시스템은 보안 이상 감지의 기반입니다.

**Karpathy:** 제가 보기에 AI가 즉시 가치를 만들 수 있는 영역은 세 가지입니다.

```
1. 자연어 검색 (Natural Language Search)
   - 마스터 데이터, 프로젝트 리소스를 자연어로 검색
   - "지난달 등록된 신규 코드 중 승인 대기 상태인 것" 같은 쿼리
   - Embedding + Vector DB 기반

2. 이상 접근 탐지 (Anomaly Access Detection)
   - 권한 시스템의 접근 로그를 분석
   - 평소와 다른 시간대/IP/패턴의 접근을 탐지
   - Isolation Forest, Autoencoder 기반

3. PRD/TRD 초안 자동 생성
   - 기존 문서 패턴을 학습하여 새 프로젝트의 초안을 생성
   - LLM + RAG (Retrieval-Augmented Generation)
```

**Ng:** 하지만 중요한 건, AI 기능을 Phase 1에 넣으면 안 됩니다. **데이터가 먼저 쌓여야 AI가 의미 있습니다.** Phase 1에서는 데이터 파이프라인을 깔아놓고, Phase 3~4에서 AI 기능을 올리는 순서가 맞습니다.

**Beck:** 좋습니다. AI 관점의 리뷰 항목을 정리해주세요.

**Ng:** 네, 6가지로 정리했습니다.

### AI 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **AI-1** | 자연어 검색 파이프라인 | 마스터 데이터 + 프로젝트 리소스에 대한 Embedding 생성 파이프라인 설계. Vector DB(Milvus/Pinecone) 선택 및 인덱싱 전략. 검색 결과의 권한 필터링(Post-filtering vs Pre-filtering). |
| **AI-2** | 이상 접근 탐지 모델 | 접근 로그 수집 포맷 표준화(누가, 언제, 어디서, 무엇에, 어떤 결과). 베이스라인 학습 기간(최소 3개월). 알림 채널 및 대응 워크플로우. |
| **AI-3** | LLM 통합 아키텍처 | LLM API 호출 시 토큰 비용 관리. Rate Limiting 및 Fallback(GPT-4 -> Claude -> 로컬 모델). Prompt 버전 관리 및 A/B 테스트 파이프라인. |
| **AI-4** | 데이터 파이프라인 선행 설계 | AI를 Phase 3~4에 배치하더라도 Phase 1부터 데이터 수집 파이프라인 구축 필요. Event Sourcing 또는 CDC(Change Data Capture)로 모든 변경 이력 보존. |
| **AI-5** | AI 추천 시스템 | 사용자별 자주 사용하는 메뉴, 검색어, 데이터 패턴을 기반으로 개인화 추천. Cold Start 문제 대응(역할 기반 기본 추천). |
| **AI-6** | AI 윤리 및 거버넌스 | AI 의사결정의 설명 가능성(Explainability). 편향(Bias) 모니터링. 사용자 동의 및 데이터 처리 투명성. |

---

## 1-2. DB 관점 (Martin Kleppmann)

**Kleppmann:** 감사합니다. 제 핵심 관점은 하나입니다. **데이터 모델이 애플리케이션을 지배합니다.** 이 시스템에서 가장 위험한 부분은 각 프로젝트가 **이질적인 데이터 모델**을 가지고 있다는 것입니다.

예를 들어볼까요?

```
프로젝트 A의 "부서" 테이블:
  - dept_id: VARCHAR(10)
  - dept_name: VARCHAR(100)
  - parent_dept_id: VARCHAR(10)

프로젝트 B의 "조직" 테이블:
  - org_code: INT
  - org_nm: NVARCHAR(200)
  - upper_org_code: INT
  - org_level: INT

같은 "부서/조직" 개념인데, 컬럼명도 다르고, 데이터 타입도 다르고,
계층 구조 표현 방식도 다릅니다 (parent_id vs level).
```

이걸 포털에서 통합 검색하거나, 권한 시스템에서 "부서 기반 접근 제어"를 하려면, **Canonical Data Model(표준 데이터 모델)**이 반드시 선행되어야 합니다. 거버넌스 없이 기술만으로 해결하려 하면 반드시 실패합니다.

**Beck:** 구체적인 검토 항목은요?

### DB 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **DB-1** | 마스터 데이터 표준 모델 | 공통 코드(코드 그룹/코드), 조직(부서/팀/직급), 사용자(프로필/연락처/소속) 각각의 Canonical Schema 정의. 코드 체계: 계층형(트리) vs 플랫(태그) vs 혼합. 변경 이력 관리 방식: SCD Type 2(Slowly Changing Dimension) 적용 여부. |
| **DB-2** | 데이터 동기화 전략 | 각 프로젝트 DB와 MDM Hub 간 동기화 방식: CDC(Debezium) vs API Polling vs Event-Driven. 충돌 해결 정책: Master Wins / Last-Write-Wins / Manual Merge. 동기화 주기: 실시간 vs 준실시간(1분) vs 배치(1시간). Eventual Consistency 허용 범위 정의. |
| **DB-3** | 권한 데이터 모델 | RBAC 기본 엔티티: User, Role, Permission, Resource. 프로젝트 스코핑: ProjectMembership(user_id, project_id, role_id). 리소스 계층: Organization > Project > Module > Feature > Data Row. 위임(Delegation) 모델: 특정 기간 동안 다른 사용자에게 권한 위임. |
| **DB-4** | 감사 로그 저장소 | 불변(Immutable) 저장소 선택: Append-only PostgreSQL, OpenSearch, ClickHouse. 보존 기간 정책: Hot(3개월, 빠른 쿼리) / Warm(1년, 압축) / Cold(5년, 아카이브). 로그 스키마: 누가(who), 언제(when), 어디서(where), 무엇을(what), 어떻게(how), 결과(result). |

---

## 1-3. 아키텍처 관점 (Martin Fowler)

**Fowler:** 제 원칙은 **"Integration at the edge, autonomy at the core"**입니다. 각 프로젝트는 내부적으로 자율적이어야 하고, 통합은 경계면에서만 일어나야 합니다.

이 시스템에서 통합이 필요한 **경계면(Boundary)**은 정확히 세 가지입니다.

```
경계면 1: Frontend 통합 (포털)
  - 사용자는 하나의 포털에서 여러 프로젝트에 접근
  - 각 프로젝트는 독립적인 Frontend 앱 (Micro Frontend)
  - 통합 지점: Shell App (레이아웃, 인증, 네비게이션)

경계면 2: 인증/인가 통합 (권한 관리)
  - 모든 프로젝트가 동일한 인증 시스템 사용 (SSO)
  - 인가(Authorization)는 중앙 정책 + 프로젝트별 세부 정책
  - 통합 지점: Auth Service + Policy Engine

경계면 3: 데이터 통합 (마스터 데이터)
  - 공통 데이터(조직, 코드, 사용자)를 중앙에서 관리
  - 각 프로젝트는 자체 비즈니스 데이터를 독립적으로 관리
  - 통합 지점: MDM Hub + API Gateway
```

핵심은 이 세 경계면 외에는 프로젝트 간 직접 의존을 만들면 안 된다는 것입니다. 프로젝트 A가 프로젝트 B의 API를 직접 호출하는 순간, 시스템은 스파게티가 됩니다.

### 아키텍처 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **ARCH-1** | Micro Frontend 아키텍처 | Shell App의 책임 범위 정의(인증, 레이아웃, 라우팅, 이벤트 버스). Remote App 계약(Contract): bootstrap/mount/unmount 생명주기. 통합 방식 선택: Module Federation vs Single-SPA vs iframe. 공유 디펜던시 전략: 어떤 라이브러리를 공유하고, 어떤 것을 각자 번들링할지. |
| **ARCH-2** | 인증/인가 아키텍처 | SSO 프로토콜 선택: OIDC + PKCE. Token 전략: Access Token(JWT, 15분) + Refresh Token(Opaque, 7일). 인가 아키텍처: API Gateway 레벨 vs Service 레벨 vs 양쪽 모두. 세션 관리: Sliding Window vs Fixed Expiry. |
| **ARCH-3** | API 아키텍처 | API Gateway 역할: 라우팅, Rate Limiting, 인증 검증, 로깅. 내부 API 표준: REST(OpenAPI 3.1) 또는 gRPC. 버전 관리: URL Path(/v1/) vs Header(Accept-Version). BFF(Backend For Frontend) 패턴 적용 여부: 포털 전용 API 계층. |
| **ARCH-4** | 장애 격리 및 복원 | Circuit Breaker 패턴: 프로젝트 A 장애가 포털 전체에 영향 주지 않도록. Bulkhead 패턴: 프로젝트별 리소스 풀 격리. Timeout/Retry 정책: 일관된 기본값 + 서비스별 커스텀. Graceful Degradation: MDM Hub 장애 시 로컬 캐시 사용. |

---

## 1-4. SW 관점 (Uncle Bob)

**Uncle Bob:** 제가 볼 때 이 시스템의 성패는 **Contract-First 설계**에 달려 있습니다. 6개 이상의 프로젝트가 하나의 포털에 통합되는데, 계약 없이 구현부터 시작하면 2개월 뒤에는 아무도 통합을 할 수 없게 됩니다.

제 제안은 세 가지입니다.

**첫째, Contract-First.** API든, 이벤트든, UI 컴포넌트든, 구현 전에 계약을 먼저 정의합니다. OpenAPI 스펙을 먼저 쓰고, 거기서 클라이언트/서버 코드를 생성합니다.

**둘째, SDK 제공.** 각 프로젝트가 Auth, Master Data, Event Bus를 직접 구현하지 않도록 **공통 SDK**를 제공합니다. SDK가 없으면 각 프로젝트마다 인증 처리 로직이 6개씩 생깁니다.

**셋째, 자동화 테스트 3대 원칙.**

```
1. Contract Test: SDK와 서비스 간 계약이 깨지지 않았는지 자동 검증
   - Pact 또는 Spring Cloud Contract 사용
   - 모든 PR에서 실행

2. Integration Test: Shell App + Remote App 통합이 동작하는지
   - Playwright로 E2E 테스트
   - 주요 사용자 시나리오 10개 이상 커버

3. Chaos Test: 특정 서비스가 죽었을 때 포털이 살아있는지
   - MDM Hub 다운 시 캐시 fallback 동작 검증
   - Auth 서비스 지연 시 사용자 경험 검증
```

### SW 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **SW-1** | Contract-First 개발 프로세스 | OpenAPI 3.1 스펙 중앙 저장소(Git repo). 스펙에서 클라이언트/서버 코드 자동 생성(openapi-generator). 스펙 변경 시 Breaking Change 감지 자동화. API 호환성 정책: Semantic Versioning + Deprecation 기간(최소 2 스프린트). |
| **SW-2** | 공통 SDK 설계 | `@portal/auth-sdk`: SSO 로그인/로그아웃, 토큰 관리, 권한 체크. `@portal/master-sdk`: 공통 코드 조회, 조직 정보 조회, 캐싱. `@portal/event-sdk`: Shell-Remote 간 이벤트 발행/구독. `@portal/ui-sdk`: Design Token + 공통 UI 컴포넌트. SDK 배포: npm private registry 또는 GitHub Packages. |
| **SW-3** | 테스트 전략 | Unit Test: 각 SDK 및 서비스의 비즈니스 로직 (커버리지 80% 이상). Contract Test: Pact 기반, 모든 API 계약 검증. Integration Test: Playwright E2E, 주요 시나리오 자동화. Performance Test: k6 또는 Artillery로 부하 테스트 (목표 응답 시간 정의). |
| **SW-4** | 코드 품질 가드레일 | ESLint/Prettier 통일 설정(monorepo 루트). Husky + lint-staged: 커밋 전 자동 검사. SonarQube: 코드 품질 게이트(중복률 3% 이하, 취약점 0개). Dependency Bot: 의존성 취약점 자동 탐지(Renovate/Dependabot). |

---

## 1-5. PM 관점 (Kent Beck)

**Beck:** 제 제안은 **Tracer Bullet** 접근법입니다. 플랫폼을 먼저 만들지 마세요. 대신 하나의 프로젝트를 가장 먼저 포털에 올리면서, **끝에서 끝까지(End-to-End) 얇게 관통하는 동작하는 시스템**을 먼저 만드세요.

```
Tracer Bullet이란?
- 어둠 속에서 총을 쏠 때, 예광탄(Tracer Bullet)을 먼저 쏘면
  탄착점을 확인하고 조준을 보정할 수 있습니다.
- 소프트웨어에서는: 전체 아키텍처를 얇게 관통하는
  하나의 완전한 기능을 먼저 구현하는 것입니다.

예시:
  "사용자가 SSO로 로그인 → 포털 메인 진입 → 프로젝트 A 대시보드 열기
   → 공통 코드로 데이터 표시 → 로그아웃"

이 한 줄의 시나리오가 동작하면:
  - SSO가 동작합니다 ✓
  - Shell App이 동작합니다 ✓
  - Micro Frontend 통합이 동작합니다 ✓
  - 마스터 데이터 API가 동작합니다 ✓
  - 권한 체크가 동작합니다 ✓
```

**Fowler:** Tracer Bullet 접근에 동의합니다. 하나 추가하면, Tracer Bullet의 대상 프로젝트는 **가장 단순한 프로젝트**가 아니라, **가장 대표적인 프로젝트**여야 합니다. 그래야 발견하는 문제가 일반적입니다.

**Beck:** 정확합니다. 정리하면, **플랫폼을 먼저 만들지 말고, 프로젝트를 올리면서 플랫폼을 함께 키우세요.**

### PM 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **PM-1** | Tracer Bullet 대상 선정 | 첫 번째 통합 대상 프로젝트 선정 기준: 대표성, 팀 역량, 일정 여유. 가장 단순한 프로젝트가 아닌 가장 대표적인 프로젝트 선택. Tracer Bullet 시나리오: 로그인 → 포털 진입 → 프로젝트 앱 → 마스터 데이터 사용 → 로그아웃. |
| **PM-2** | 단계별 롤아웃 계획 | Phase 0(설계): 4주 - Contract 정의, SDK 스켈레톤, 인프라 셋업. Phase 1(Tracer): 6주 - 첫 번째 프로젝트 포털 통합. Phase 2(확장): 8주 - 2~3개 프로젝트 추가, 마스터 데이터 고도화. Phase 3(고도화): 8주 - AI 기능, 고급 권한, 감사 대시보드. Phase 4(최적화): 지속 - 성능, UX, AI 개인화. |
| **PM-3** | 이해관계자 관리 | 각 프로젝트 팀의 포털 통합 부담 최소화 전략. SDK 제공 + 마이그레이션 가이드 + 페어 프로그래밍 지원. 통합 과정에서 각 프로젝트의 독립 배포 능력 유지 보장. Go/No-Go 기준: 각 Phase 진입 전 필수 검증 항목 정의. |
| **PM-4** | 위험 관리 | 리스크 1: 프로젝트 팀의 저항 → SDK 품질과 문서로 극복. 리스크 2: 기술 스택 불일치 → Adapter 패턴으로 수용. 리스크 3: 마스터 데이터 품질 → 거버넌스 위원회 + 자동 검증. 리스크 4: 성능 저하 → 성능 버짓 설정 + 모니터링 자동화. |

---

## 1-6. UX 관점 (Jakob Nielsen)

**Nielsen:** 이 포털은 사용자의 **매일의 업무 진입점(Daily Entry Point)**입니다. 아침에 출근해서 가장 먼저 여는 화면이 이 포털입니다. 그래서 UX가 단순히 "예쁜 화면"이 아니라, **업무 효율성**의 핵심입니다.

두 가지를 강조하겠습니다.

**첫째, Design Token은 선택이 아니라 필수입니다.** 6개 프로젝트가 제각각 다른 색상, 폰트, 간격을 쓰면, 사용자는 앱을 전환할 때마다 인지 부하(Cognitive Load)를 겪습니다. "이 버튼은 저장인가? 취소인가?" 같은 혼란이 생깁니다.

```
Design Token이 없는 포털:
  프로젝트 A: 파란 버튼이 "저장", 빨간 버튼이 "삭제"
  프로젝트 B: 초록 버튼이 "확인", 파란 버튼이 "취소"
  → 사용자: "파란 버튼이 뭐였지...?"

Design Token이 있는 포털:
  모든 프로젝트: --color-primary 버튼이 "주요 액션", --color-danger 버튼이 "위험 액션"
  → 사용자: 색상만 보고 즉시 판단 가능
```

**둘째, 통합 검색은 포털의 생명줄입니다.** 6개 프로젝트에 걸쳐 있는 데이터를 하나의 검색창에서 찾을 수 있어야 합니다. Google이 "검색 하나로 모든 것"을 보여준 것처럼, 포털도 마찬가지입니다.

### UX 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **UX-1** | Design Token 표준 | 색상, 타이포그래피, 간격, 그림자, 모서리 반경의 변수 정의. 다크 모드 지원 계획(CSS 변수 기반 테마 전환). 프레임워크별 변환: CSS Variables → SCSS, Tailwind Config, Styled-Components Theme. 접근성(a11y) 기준: WCAG 2.1 AA 이상(대비율 4.5:1, 포커스 표시, 스크린 리더 지원). |
| **UX-2** | 통합 검색 UX | 검색 범위: 마스터 데이터, 프로젝트 리소스, 메뉴, 사용자, 문서. 검색 결과 표시: 카테고리별 그룹핑 + 권한 필터링(볼 수 없는 결과는 표시 안 함). 최근 검색/자주 사용 메뉴의 개인화. 키보드 단축키(Cmd/Ctrl+K)로 즉시 접근. |

---

## 1-7. 보안 관점 (Bruce Schneier)

**Schneier:** 보안은 제품이 아니라 **프로세스**입니다. 방화벽 하나 놓았다고 안전하지 않습니다. 이 시스템에서 제가 강조할 두 가지가 있습니다.

**첫째, Zero Trust.** 네트워크 경계가 아니라, 모든 요청을 검증합니다. 내부 API 호출이라도 인증 토큰을 반드시 포함해야 합니다. "내부 네트워크니까 안전하다"는 가장 위험한 가정입니다.

```
Zero Trust 원칙:
  1. 모든 요청은 인증 + 인가를 거친다 (내부 포함)
  2. 최소 권한 원칙: 필요한 것만, 필요한 시간만
  3. 접근은 지속적으로 검증된다 (세션 시작 시 1회가 아님)
  4. 모든 접근은 기록된다 (감사 로그)
```

**둘째, Audit Trail은 선택이 아니라 의무입니다.** 누가 언제 어떤 데이터에 접근했는지, 어떤 변경을 했는지, 모든 것이 기록되어야 합니다. 이것은 보안 사고 후 추적을 위해서도 필요하고, 규정 준수(Compliance)를 위해서도 필요합니다.

### 보안 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **SEC-1** | Zero Trust 구현 | 서비스 간 통신: mTLS 또는 Service Mesh(Istio) 기반 인증. API Gateway: 모든 요청에 JWT 검증 + Rate Limiting. 내부 API도 인증 필수: Service Account Token 발급 및 관리. 네트워크 세그멘테이션: 프로젝트별 네트워크 격리. |
| **SEC-2** | 감사 로그 체계 | 로그 수집: 모든 인증/인가/데이터 접근 이벤트. 로그 무결성: 변조 방지(Hash Chain 또는 WORM 스토리지). 로그 분석: 실시간 이상 탐지 + 주기적 감사 리포트. 규정 준수: 개인정보보호법, 정보보안 규정에 따른 보존 기간 및 접근 제한. |
| **SEC-3** | 취약점 관리 프로세스 | OWASP Top 10 기반 보안 점검 자동화(SAST/DAST). 의존성 취약점 스캔: Snyk, Trivy(컨테이너 이미지). 보안 사고 대응 계획(Incident Response Plan): 탐지 → 격리 → 분석 → 복구 → 사후 검토. 정기 보안 감사 및 침투 테스트 계획(분기 1회). |

---

**Beck (사회자):** 감사합니다. Session 1에서 각 전문가가 총 **27개의 검토 항목**을 제시했습니다. 이제 Session 2에서 핵심 쟁점에 대한 교차 토론을 시작합니다.

---

# Session 2: 교차 토론 - 핵심 쟁점 3가지

**Beck (사회자):** Session 1의 발표를 들으면서 **세 가지 핵심 쟁점**이 보였습니다. 이 세 가지에 대해 합의가 필요합니다.

```
쟁점 1: 권한 모델을 어떤 방식으로 설계할 것인가?
쟁점 2: 마스터 데이터를 어떻게 통합할 것인가?
쟁점 3: 다중 기술 스택의 포털을 어떻게 통합할 것인가?
```

---

## 쟁점 1: 권한 모델 - RBAC vs ABAC

**Beck:** 첫 번째 쟁점입니다. 권한 모델을 RBAC(Role-Based Access Control)로 갈지, ABAC(Attribute-Based Access Control)로 갈지. Kleppmann, 먼저 시작해주세요.

**Kleppmann:** 먼저 두 모델의 차이를 명확히 합시다.

```
RBAC (Role-Based Access Control):
  - 사용자에게 역할(Role)을 부여하고, 역할에 권한(Permission)을 매핑
  - 예: "프로젝트A_관리자" 역할 → "프로젝트A의 모든 데이터 읽기/쓰기" 권한
  - 장점: 단순, 이해하기 쉬움, 감사 용이
  - 단점: 세밀한 제어 어려움, 역할 폭발(Role Explosion) 위험

ABAC (Attribute-Based Access Control):
  - 사용자 속성, 리소스 속성, 환경 속성의 조합으로 접근 결정
  - 예: "부서=영업 AND 직급>=과장 AND 시간=업무시간 → 고객 데이터 읽기 허용"
  - 장점: 매우 세밀한 제어, 역할 폭발 없음
  - 단점: 복잡, 디버깅 어려움, 성능 우려
```

**Schneier:** 보안 관점에서는 ABAC가 이상적입니다. 하지만 현실적으로, ABAC를 처음부터 도입하면 **권한 정책을 작성하고 관리할 수 있는 사람이 없습니다.** 개발자도 어렵고, 관리자는 더 어렵습니다.

**Uncle Bob:** 동의합니다. ABAC는 강력하지만, YAGNI(You Aren't Gonna Need It) 원칙에 어긋납니다. 지금 당장 필요한 건 "이 사용자가 이 프로젝트에 접근할 수 있는가?"이지, "이 사용자의 부서가 영업이고 직급이 과장 이상이고 현재 업무 시간인가?"가 아닙니다.

**Fowler:** 하지만 순수 RBAC도 문제입니다. 6개 프로젝트가 있고, 각 프로젝트마다 관리자/편집자/뷰어 역할이 있으면, 최소 18개 역할이 생깁니다. 여기에 모듈별 세분화까지 하면 역할이 50개, 100개로 폭발합니다.

**Ng:** 제 제안은 **RBAC + Project Scope**입니다. 역할을 프로젝트에 종속시키는 것입니다.

```
역할은 5개만: Super Admin, Project Admin, Editor, Viewer, Guest
스코프로 적용:
  - 사용자 A는 "프로젝트 X에서 Editor"
  - 사용자 A는 "프로젝트 Y에서 Viewer"
  - 사용자 B는 "전체에서 Super Admin"

데이터 모델:
  ProjectMembership {
    user_id: string
    project_id: string  // 스코프
    role: Role          // 5개 중 하나
    granted_at: timestamp
    granted_by: string
    expires_at: timestamp | null  // 임시 권한 지원
  }
```

**Kleppmann:** 좋습니다. 그런데 나중에 "특정 메뉴만 접근 가능"이나 "특정 데이터 행만 볼 수 있음" 같은 세밀한 요구가 오면요?

**Fowler:** 그래서 **Policy Engine**을 추가합니다. 기본은 RBAC + Project Scope로 가되, 세밀한 정책이 필요한 경우 OPA(Open Policy Agent) 같은 정책 엔진으로 확장합니다. 중요한 건 RBAC가 90%의 케이스를 처리하고, OPA가 나머지 10%의 예외를 처리하는 구조입니다.

```
권한 판단 흐름:

요청 → [1] RBAC 체크 (빠름, 캐시 가능)
         |
         ├── 명확한 허용/거부 → 완료
         |
         └── 추가 정책 필요 → [2] OPA 정책 엔진 (유연, 약간 느림)
                                  |
                                  └── 정책 평가 → 허용/거부
```

**Uncle Bob:** 이 구조면 충분합니다. RBAC로 시작해서 OPA로 확장할 수 있으니, Phase 1에서는 RBAC만 구현하고, 세밀한 정책이 필요해지는 Phase 2~3에서 OPA를 붙이면 됩니다.

**Schneier:** 한 가지 추가합니다. **권한 변경 감사(Audit)**가 반드시 포함되어야 합니다. "누가 누구에게 어떤 역할을 부여/회수했는지"가 기록되어야 합니다. 이건 보안의 기본입니다.

**Karpathy:** 그리고 AI 관점에서, 이 권한 변경 로그에 이상 패턴이 발생하면 알림을 주는 기능을 Phase 3에서 추가할 수 있습니다. 예를 들어, 퇴사 예정자에게 대량의 권한이 부여되는 패턴 같은 것을 탐지하는 것입니다.

> **합의:** RBAC + Project Scope를 기본으로 하되, 세밀한 정책이 필요한 경우 Policy Engine(OPA)으로 확장. 역할은 5개로 제한(Super Admin, Project Admin, Editor, Viewer, Guest). 모든 권한 변경은 감사 로그에 기록.

---

## 쟁점 2: 마스터 데이터 통합 전략

**Beck:** 두 번째 쟁점입니다. 각 프로젝트가 가진 마스터 데이터(조직, 코드, 사용자)를 어떻게 통합할 것인가. Kleppmann, 리드해주세요.

**Kleppmann:** 마스터 데이터 통합 전략에는 크게 세 가지 선택지가 있습니다.

```
선택지 1: 중앙 집중형 MDM (Master Data Management)
  - 하나의 MDM Hub가 모든 마스터 데이터의 Golden Source
  - 모든 프로젝트는 MDM Hub에서만 읽기
  - 장점: 데이터 일관성 보장
  - 단점: Single Point of Failure, 각 프로젝트의 자율성 제한

선택지 2: 분산형 (Data Mesh 스타일)
  - 각 프로젝트가 자기 데이터의 주인
  - 프로젝트 간 데이터 공유는 이벤트/API로
  - 장점: 자율성, 장애 격리
  - 단점: 데이터 불일치, 거버넌스 어려움

선택지 3: 하이브리드 (MDM Hub + 로컬 캐시)
  - MDM Hub가 Golden Source이지만, 각 프로젝트는 로컬 캐시 보유
  - MDM Hub 변경 시 이벤트로 캐시 갱신
  - 장점: 일관성 + 가용성 둘 다
  - 단점: 구현 복잡도
```

**Fowler:** 선택지 3이 정답입니다. 하지만 "하이브리드"라고 말하면 모든 게 하이브리드이니, 좀 더 구체적으로 해야 합니다. 제 제안은 이렇습니다:

```
MDM Hub (Golden Source)
    │
    ├── REST API: 실시간 조회 (CRUD)
    │     └── 캐시: Redis, TTL 5분
    │
    ├── Event Bus: 변경 알림 (Kafka/RabbitMQ)
    │     └── 각 프로젝트가 구독하여 로컬 캐시 갱신
    │
    └── Bulk Sync: 초기 데이터 로딩 및 정기 전체 동기화
          └── 일 1회 배치 (정합성 검증 겸)
```

**Uncle Bob:** 여기서 핵심 질문입니다. MDM Hub의 API가 죽으면 각 프로젝트는 어떻게 되나요?

**Fowler:** 바로 그래서 **Circuit Breaker**가 필요합니다. SDK에 내장합니다.

```typescript
// @portal/master-sdk 내부 로직
class MasterDataClient {
  private circuitBreaker: CircuitBreaker;
  private localCache: Map<string, CachedData>;

  async getCode(groupId: string, codeId: string): Promise<CodeInfo> {
    try {
      // 1차: MDM Hub API 호출 (Circuit Breaker 보호)
      return await this.circuitBreaker.execute(() =>
        this.api.getCode(groupId, codeId)
      );
    } catch (error) {
      // Circuit Open 또는 API 장애
      // 2차: 로컬 캐시에서 조회
      const cached = this.localCache.get(`${groupId}:${codeId}`);
      if (cached && !cached.isExpired()) {
        return cached.data;  // 캐시 데이터 반환 (stale 가능)
      }
      throw new MasterDataUnavailableError(groupId, codeId);
    }
  }
}
```

**Kleppmann:** 좋습니다. 그런데 **거버넌스**도 중요합니다. 누가 마스터 데이터의 표준을 정하고, 누가 변경을 승인하는가. 기술만으로는 해결이 안 됩니다.

```
마스터 데이터 거버넌스 구조:

1. Data Steward (데이터 관리자): 각 데이터 도메인별 담당자
   - 코드 데이터 관리자: 코드 체계 정의/변경 승인
   - 조직 데이터 관리자: 조직 구조 변경 승인
   - 사용자 데이터 관리자: 사용자 정보 정책

2. Data Quality Rule (품질 규칙):
   - 필수 필드 검증
   - 코드 형식 검증 (정규식)
   - 참조 무결성 검증
   - 중복 탐지

3. 변경 승인 워크플로우:
   - 요청 → 검증(자동) → 승인(Data Steward) → 적용 → 알림
```

**Nielsen:** 여기서 UX 관점을 넣겠습니다. 마스터 데이터 관리 화면은 **비전문가도 쓸 수 있어야** 합니다. 현업 담당자가 직접 코드를 등록하고 관리할 수 있어야 합니다. 엑셀 업로드/다운로드는 기본이고, 변경 이력을 쉽게 확인할 수 있어야 합니다.

**Ng:** AI를 접목하면, 마스터 데이터 품질 검증을 자동화할 수 있습니다. 예를 들어, 신규 등록된 코드가 기존 코드와 유사한데 다른 값을 가지면 경고를 주는 것입니다. "이 코드와 유사한 기존 코드가 있습니다: [코드 A]. 통합하시겠습니까?"

> **합의:** MDM Hub(Golden Source) + REST API + Event Bus를 통한 로컬 캐시 갱신. SDK에 Circuit Breaker 내장. 데이터 거버넌스 체계(Data Steward, 품질 규칙, 승인 워크플로우) 필수 수립. Phase 3에서 AI 기반 품질 검증 추가.

---

## 쟁점 3: 다중 기술 스택 포털 통합

**Beck:** 세 번째 쟁점입니다. React, Vue, Angular 등 다양한 기술 스택을 하나의 포털에 어떻게 통합하는가. Fowler, 리드해주세요.

**Fowler:** 이미 Session 1에서 Micro Frontend를 언급했는데, 구체적인 통합 방식을 정해야 합니다. 핵심 질문은 **Shell App과 Remote App 간의 계약(Contract)**을 어떻게 정의하느냐입니다.

제 제안은 **Module Federation을 기본으로 하되, 프레임워크 독립적인 계약을 정의**하는 것입니다.

```typescript
// Shell App이 Remote App에 요구하는 계약
interface RemoteAppContract {
  // 생명주기
  bootstrap(): Promise<void>;
  mount(container: HTMLElement, context: ShellContext): Promise<void>;
  unmount(): Promise<void>;

  // 메타데이터
  manifest: AppManifest;
}

interface AppManifest {
  id: string;                    // "project-a"
  name: string;                  // "프로젝트 A"
  version: string;               // "1.2.0"
  basePath: string;              // "/project-a"
  entry: string;                 // remoteEntry.js URL
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

**Uncle Bob:** 이 계약을 **npm 패키지로 배포**해야 합니다. `@portal/shell-contracts`라는 패키지에 TypeScript 인터페이스를 담아서, 모든 Remote App이 이 인터페이스를 구현하도록 강제합니다.

**Karpathy:** 프레임워크별로 **어댑터(Adapter)**를 제공하면 좋겠습니다. React 프로젝트는 React 어댑터를, Vue 프로젝트는 Vue 어댑터를 쓰면 됩니다.

```typescript
// @portal/react-adapter
export function createReactRemoteApp(
  App: React.ComponentType<{ context: ShellContext }>,
  manifest: AppManifest
): RemoteAppContract {
  let root: ReactDOM.Root | null = null;

  return {
    manifest,
    async bootstrap() {
      // React-specific initialization
    },
    async mount(container, context) {
      root = ReactDOM.createRoot(container);
      root.render(<App context={context} />);
    },
    async unmount() {
      root?.unmount();
      root = null;
    },
  };
}

// 사용 예시 (프로젝트 A)
import { createReactRemoteApp } from '@portal/react-adapter';
import { ProjectAApp } from './App';
import { manifest } from './manifest';

export default createReactRemoteApp(ProjectAApp, manifest);
```

**Nielsen:** 여기서 중요한 건 **Design Token**입니다. 프레임워크가 달라도 시각적으로 통일되어야 합니다. CSS 변수 기반 Design Token은 프레임워크에 독립적이므로, 모든 Remote App이 동일한 Token 파일을 import하면 됩니다.

```
통합 전략 요약:

1. Contract: @portal/shell-contracts (TypeScript 인터페이스)
2. Adapter: @portal/react-adapter, @portal/vue-adapter, @portal/angular-adapter
3. Design: @portal/design-tokens (CSS 변수, SCSS, Tailwind Config)
4. SDK: @portal/auth-sdk, @portal/master-sdk, @portal/event-sdk
```

**Fowler:** 그리고 **공유 디펜던시(Shared Dependencies)** 전략도 정해야 합니다. Module Federation에서 어떤 라이브러리를 공유할지:

```
공유 대상 (singleton):
  - react, react-dom (React 프로젝트 간)
  - @portal/auth-sdk
  - @portal/master-sdk
  - @portal/event-sdk
  - @portal/design-tokens

공유하지 않는 것:
  - 프로젝트별 비즈니스 로직 라이브러리
  - 프로젝트별 상태 관리 (Redux, Zustand 등)
  - 프로젝트별 API 클라이언트
```

> **합의:** Micro Frontend(Module Federation 기본) + 프레임워크 독립적 Contract(@portal/shell-contracts) + 프레임워크별 Adapter + Design Token(CSS 변수) + 공유 SDK. 공유 디펜던시는 Shell 관련 SDK만, 나머지는 각 프로젝트 자율.

---

**Beck (사회자):** Session 2에서 세 가지 핵심 쟁점에 대한 합의를 도출했습니다. 정리하면:

| 쟁점 | 합의 |
|------|------|
| 권한 모델 | RBAC + Project Scope + Policy Engine(OPA) |
| 마스터 데이터 통합 | MDM Hub + API + 로컬 캐시 + Circuit Breaker + 거버넌스 |
| 다중 기술 스택 통합 | Micro Frontend(Module Federation) + 공통 계약 + Design Token |

---

# Session 3: 최종 종합 - PRD/TRD 체크리스트

**Beck (사회자):** Session 1과 2의 논의를 종합하여, PRD와 TRD에 반드시 포함되어야 하는 체크리스트를 만듭시다. 그리고 우선순위 로드맵도 확정합시다.

---

## 3-1. PRD 필수 검토 항목

**Beck:** PRD(Product Requirements Document)에 반드시 들어가야 할 10가지 항목입니다. 각 전문가의 의견을 반영해서 정리합니다.

**Nielsen:** 사용자 관점에서 PRD는 "왜 만드는가"와 "무엇을 만드는가"를 명확히 해야 합니다.

| # | PRD 항목 | 설명 | 담당 관점 |
|---|----------|------|-----------|
| P-1 | 프로젝트 개요 및 비전 | 다중 프로젝트 포털의 존재 이유와 비즈니스 가치 | PM |
| P-2 | 사용자 페르소나 | 최소 3종: 일반 사용자, 프로젝트 관리자, 시스템 관리자 | UX |
| P-3 | 핵심 사용자 시나리오 | 로그인→포털→프로젝트 접근→마스터 데이터 활용→로그아웃 E2E 시나리오 | UX/PM |
| P-4 | 기능 요구사항 (포털) | Shell App, Remote App 통합, 통합 검색, 알림, 메뉴 관리 | 아키텍처 |
| P-5 | 기능 요구사항 (권한 관리) | RBAC + Project Scope, 역할 관리 화면, 위임, 감사 로그 | 보안/DB |
| P-6 | 기능 요구사항 (마스터 데이터) | 코드 관리, 조직 관리, 사용자 관리, 변경 승인, 엑셀 연동 | DB/UX |
| P-7 | 비기능 요구사항 | 성능(응답 시간 기준), 가용성(99.9%), 보안 등급, 접근성(WCAG AA) | 전체 |
| P-8 | 통합 요구사항 | 기존 프로젝트 통합 방법, SSO 연동, 외부 시스템 인터페이스 | 아키텍처 |
| P-9 | 성공 지표(KPI) | 포털 활성 사용률, 평균 앱 전환 시간, 마스터 데이터 정합성 비율 | PM |
| P-10 | 제약 조건 및 가정 | 기존 시스템 제약, 기술 스택 제한, 조직 제약, 일정 | PM |

---

## 3-2. TRD 필수 검토 항목

**Uncle Bob:** TRD(Technical Requirements Document)는 AI가 코드를 생성할 수 있을 만큼 구체적이어야 합니다. 13가지 필수 항목입니다.

| # | TRD 항목 | 설명 | 담당 관점 |
|---|----------|------|-----------|
| T-1 | 시스템 아키텍처 다이어그램 | 전체 컴포넌트 배치도, 통신 흐름, 기술 스택 명시 | 아키텍처 |
| T-2 | Micro Frontend 설계 | Shell App 구조, Remote App 계약, Module Federation 설정, 라우팅 | 아키텍처/SW |
| T-3 | 인증/인가 설계 | SSO(OIDC+PKCE) 흐름, JWT 구조, RBAC 모델, OPA 정책 예시 | 보안/DB |
| T-4 | 마스터 데이터 설계 | Canonical Schema, API 명세, 동기화 전략, Circuit Breaker 설계 | DB |
| T-5 | API 설계 | OpenAPI 3.1 스펙, 엔드포인트 목록, 입출력 스키마, 에러 코드 체계 | SW |
| T-6 | 데이터베이스 설계 | ERD, 테이블 정의서, 인덱스 전략, 파티셔닝 전략 | DB |
| T-7 | SDK 설계 | 각 SDK의 API, 내부 구조, 의존성, 사용 예시 코드 | SW |
| T-8 | Design System 설계 | Design Token 정의, 컴포넌트 목록, Storybook 구조 | UX |
| T-9 | 인프라/배포 설계 | 컨테이너 구조, CI/CD 파이프라인, 환경별 설정, 모니터링 | 아키텍처 |
| T-10 | 보안 설계 | Zero Trust 구현, 감사 로그 체계, 취약점 관리, 암호화 전략 | 보안 |
| T-11 | 테스트 전략 | Unit/Contract/Integration/E2E/Performance 테스트 계획 | SW |
| T-12 | 디렉토리 구조 | 모노레포 구조, 패키지 구성, 파일 명명 규칙 | SW |
| T-13 | 환경 변수 및 설정 | 환경별(.env) 변수 목록, Feature Flags, 외부 서비스 연결 정보 | 아키텍처/SW |

---

## 3-3. 우선순위 로드맵

**Beck:** 이제 Phase 0~4 로드맵을 확정합시다. 각 Phase의 범위와 기간, 산출물을 명확히 합니다.

| Phase | 이름 | 기간 | 핵심 목표 | 주요 산출물 |
|-------|------|------|-----------|-------------|
| **Phase 0** | 설계 및 기반 | 4주 | Contract 정의, 인프라 셋업, SDK 스켈레톤 | OpenAPI 스펙, Shell App 스켈레톤, CI/CD 파이프라인, Design Token v1, SDK 인터페이스 정의 |
| **Phase 1** | Tracer Bullet | 6주 | 첫 번째 프로젝트 E2E 통합 | SSO 로그인 동작, Shell + 1개 Remote App 통합, 기본 RBAC 동작, 마스터 데이터 API(코드/조직) 동작, 기본 감사 로그 |
| **Phase 2** | 확장 | 8주 | 2~3개 프로젝트 추가, 기능 고도화 | 추가 Remote App 통합, 마스터 데이터 관리 화면, 권한 관리 화면, 통합 검색 v1, 엑셀 연동, 알림 시스템 |
| **Phase 3** | 고도화 | 8주 | AI 기능, 고급 권한, 모니터링 | 자연어 검색, 이상 접근 탐지 v1, OPA 정책 엔진 통합, 감사 대시보드, 성능 최적화 |
| **Phase 4** | 최적화 | 지속 | 개인화, 자동화, 운영 성숙 | AI 개인화 추천, PRD/TRD 자동 초안, A/B 테스트 인프라, 셀프서비스 프로젝트 온보딩 |

**Fowler:** 각 Phase 사이에 **Go/No-Go 게이트**가 있어야 합니다. Phase 1 완료 후 "이 아키텍처가 확장 가능한가?"를 검증하고 나서 Phase 2로 넘어가야 합니다.

**Beck:** 맞습니다. Go/No-Go 기준을 정의합시다.

```
Phase 0 → Phase 1 Go 기준:
  □ OpenAPI 스펙 최소 3개 서비스 완료
  □ Shell App 스켈레톤이 로컬에서 구동
  □ CI/CD 파이프라인으로 스테이징 배포 가능
  □ Design Token이 Storybook에서 확인 가능

Phase 1 → Phase 2 Go 기준:
  □ Tracer Bullet 시나리오 E2E 성공
  □ 첫 번째 Remote App이 포털에서 정상 동작
  □ SSO 로그인/로그아웃 정상 동작
  □ 마스터 데이터 API 응답 시간 200ms 이하
  □ 기본 RBAC로 권한 체크 정상 동작

Phase 2 → Phase 3 Go 기준:
  □ 최소 3개 프로젝트 포털 통합 완료
  □ 마스터 데이터 관리 화면으로 CRUD 가능
  □ 권한 관리 화면으로 역할 부여/회수 가능
  □ 통합 검색에서 프로젝트 간 결과 표시
  □ 전체 E2E 테스트 통과율 95% 이상

Phase 3 → Phase 4 Go 기준:
  □ AI 검색 정확도(MRR@10) 0.7 이상
  □ 이상 접근 탐지 False Positive 비율 10% 이하
  □ 감사 대시보드에서 30일 내 모든 접근 조회 가능
  □ P95 응답 시간 500ms 이하
```

> **합의:** Phase 0~4 로드맵 확정. 각 Phase 간 Go/No-Go 게이트로 품질 보장.

---

# Session 4: 누락된 관점 보완 토론

**Beck (사회자):** Session 1~3에서 핵심 아키텍처와 로드맵을 잡았습니다. 이제 **놓친 관점이 없는지** 보완 토론을 합시다. 제가 보기에 6가지 영역이 아직 다뤄지지 않았습니다.

```
1. DevOps / Observability
2. 개발자 경험(DX)
3. 데이터 거버넌스 심화
4. 성능(Performance) 상세
5. 마이그레이션 전략
6. 다국어(i18n) / 멀티 테넌시
```

---

## 4-1. DevOps / Observability (OPS-1~4)

**Beck:** 먼저 DevOps와 Observability입니다. Fowler, 이 부분을 짚어주세요.

**Fowler:** Micro Frontend + Microservice 아키텍처에서는 **관측 가능성(Observability)**이 생명줄입니다. 단일 요청이 Shell App → API Gateway → Auth Service → MDM Hub → 프로젝트 API를 거치는데, 어디서 느려지는지, 어디서 에러가 나는지를 추적할 수 없으면 운영이 불가능합니다.

**Uncle Bob:** 동의합니다. 세 가지 기둥(Three Pillars)이 필요합니다.

```
Observability 3대 기둥:

1. Logs (구조화 로그)
   - JSON 형식, 일관된 스키마
   - 필수 필드: timestamp, service, level, trace_id, user_id, message
   - 수집: Fluentd/Vector → OpenSearch/Loki

2. Metrics (메트릭)
   - RED 메트릭: Rate(초당 요청), Errors(에러율), Duration(응답 시간)
   - 각 서비스별 + API 엔드포인트별 수집
   - 수집: Prometheus + Grafana

3. Traces (분산 추적)
   - 요청 ID(Trace ID)로 전체 호출 체인 추적
   - 서비스 간 전파: W3C Trace Context 헤더
   - 수집: OpenTelemetry → Jaeger/Tempo
```

**Schneier:** 보안 관점에서 하나 추가합니다. **보안 이벤트 모니터링**도 Observability에 포함되어야 합니다. 인증 실패, 권한 거부, 비정상 접근 패턴을 실시간으로 모니터링하고 알림을 받아야 합니다.

### DevOps/Observability 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **OPS-1** | CI/CD 파이프라인 설계 | 모노레포 전략: Turborepo/Nx로 변경된 패키지만 빌드/배포. 배포 전략: Blue-Green 또는 Canary 배포. Shell App과 Remote App의 독립 배포 파이프라인. 환경: dev → staging → production 3단계. |
| **OPS-2** | 구조화 로그 표준 | JSON 로그 스키마 표준화. 모든 서비스가 동일한 로그 필드 사용. 로그 레벨 정책: ERROR(즉시 알림), WARN(대시보드), INFO(분석용). 민감 정보 마스킹(PII, 토큰 등). |
| **OPS-3** | 분산 추적(Distributed Tracing) | OpenTelemetry SDK를 모든 서비스에 내장. W3C Trace Context 헤더로 서비스 간 전파. Frontend에서도 Trace 시작(Browser → API 요청까지 연결). Trace 샘플링 전략: Head-based 10% + Error 100%. |
| **OPS-4** | 알림 및 온콜 체계 | 알림 채널: Slack/Teams + PagerDuty/OpsGenie. 알림 등급: P1(서비스 다운, 5분 이내 대응) ~ P4(개선 사항, 다음 스프린트). Runbook: 주요 장애 시나리오별 대응 절차 문서화. SLO/SLI 정의: 가용성 99.9%, P95 응답 500ms, 에러율 0.1% 이하. |

---

## 4-2. 개발자 경험 - DX (DX-1~3)

**Beck:** 다음은 개발자 경험입니다. 이 시스템을 사용하는 개발자는 두 종류입니다: 포털 플랫폼 개발자와, 각 프로젝트에서 포털에 앱을 올리는 개발자. 특히 후자의 경험이 중요합니다.

**Uncle Bob:** 각 프로젝트 개발자가 포털에 앱을 올리기 위해 해야 할 일이 10단계면, 그 프로젝트는 포털을 안 씁니다. **3단계 이내**로 줄여야 합니다.

```
이상적인 프로젝트 온보딩 과정:

Step 1: SDK 설치
  npm install @portal/react-adapter @portal/auth-sdk @portal/master-sdk

Step 2: Adapter로 앱 감싸기
  export default createReactRemoteApp(MyApp, myManifest);

Step 3: 포털에 등록
  portal-cli register --manifest ./manifest.json

완료. 이 3단계면 프로젝트 앱이 포털에 표시됩니다.
```

**Karpathy:** 여기에 CLI 도구와 **템플릿(Scaffold)**를 제공하면 더 좋습니다.

```bash
# 새 프로젝트 템플릿 생성
portal-cli create my-project --template react

# 생성되는 파일:
# my-project/
# ├── src/
# │   ├── App.tsx          # 기본 앱 컴포넌트
# │   ├── remote-entry.ts  # Module Federation 엔트리
# │   └── manifest.json    # 포털 등록 메타데이터
# ├── webpack.config.js    # Module Federation 설정 포함
# ├── package.json         # SDK 의존성 포함
# └── README.md            # 가이드
```

**Nielsen:** 그리고 **개발자 포털(Developer Portal)** 화면이 있어야 합니다. API 문서, SDK 가이드, 디자인 시스템 Storybook, 트러블슈팅 FAQ를 한 곳에서 볼 수 있는 곳이요.

### DX 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **DX-1** | CLI 도구 (portal-cli) | 프로젝트 스캐폴딩: `portal-cli create <name> --template <react\|vue\|angular>`. 로컬 개발: `portal-cli dev` (Shell App 에뮬레이션 모드). 포털 등록: `portal-cli register --manifest <path>`. 배포: `portal-cli deploy --env <staging\|production>`. |
| **DX-2** | 개발자 문서 포털 | API 문서: Swagger UI (OpenAPI 스펙 기반 자동 생성). SDK 가이드: 설치, 초기화, 사용 예시 코드. Design System: Storybook으로 컴포넌트 카탈로그. 트러블슈팅: FAQ + 에러 코드 사전. |
| **DX-3** | 로컬 개발 환경 | Shell App 없이 Remote App을 독립적으로 개발/테스트 가능. Mock Auth Context: 로컬에서 인증 없이 개발. Mock Master Data: 로컬에서 MDM Hub 없이 개발. Hot Reload: 코드 변경 시 즉시 반영. |

---

## 4-3. 데이터 거버넌스 심화 (GOV-1~3)

**Beck:** 데이터 거버넌스를 좀 더 깊이 파봅시다. Kleppmann, 리드해주세요.

**Kleppmann:** Session 2에서 기본 거버넌스를 다뤘지만, 몇 가지 빠진 것이 있습니다.

**첫째, 데이터 리니지(Data Lineage)입니다.** 마스터 데이터가 어디서 생성되어, 어디로 전파되고, 어디서 사용되는지를 추적할 수 있어야 합니다. "이 코드 값을 변경하면 어떤 프로젝트에 영향을 주는가?"를 알 수 있어야 합니다.

**둘째, 데이터 분류(Classification)입니다.** 모든 데이터 필드에 대해 민감도 등급을 분류해야 합니다.

```
데이터 분류 등급:

Level 1 (공개): 시스템 코드명, 메뉴 이름 등
Level 2 (내부): 조직 구조, 프로젝트 목록 등
Level 3 (기밀): 개인 연락처, 급여 정보 등
Level 4 (극비): 인사 평가, 보안 감사 결과 등

각 등급별:
  - 접근 권한 기준
  - 로그 기록 수준
  - 암호화 요구사항
  - 보존/파기 정책
```

**Schneier:** 데이터 분류는 보안의 핵심입니다. Level 3 이상은 **반드시 암호화 저장**이고, 접근 시 감사 로그를 남겨야 합니다.

### 데이터 거버넌스 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **GOV-1** | 데이터 리니지 추적 | 마스터 데이터 원천(Source of Truth) 매핑. 데이터 흐름: 생성 → 전파 → 소비 경로 문서화. 영향도 분석: 특정 마스터 데이터 변경 시 영향받는 시스템/프로젝트 자동 파악. 변경 전 영향도 리포트 제공. |
| **GOV-2** | 데이터 분류 및 보호 | 4단계 분류 체계(공개/내부/기밀/극비) 수립. 필드 레벨 분류: 각 테이블의 각 컬럼에 분류 등급 태깅. 등급별 접근 제어, 암호화, 마스킹, 감사 정책 정의. 정기 분류 검토(분기 1회). |
| **GOV-3** | 데이터 품질 관리 | 품질 지표 정의: 완전성(Completeness), 정확성(Accuracy), 일관성(Consistency), 적시성(Timeliness). 자동 품질 검사: 등록/변경 시 규칙 기반 검증. 품질 대시보드: 데이터 도메인별 품질 점수 시각화. 이상 데이터 알림: 품질 기준 미달 시 Data Steward에게 알림. |

---

## 4-4. 성능(Performance) 상세 (PERF-1~3)

**Beck:** 성능 요구사항을 구체화합시다. "빨라야 한다"가 아니라, **숫자로** 정의해야 합니다.

**Nielsen:** UX 관점에서 성능 기준을 제시합니다.

```
Jakob Nielsen의 응답 시간 기준:
  - 0.1초 이내: 사용자는 즉시 반응이라고 느낌
  - 1초 이내: 사용자의 사고 흐름은 유지되지만 지연을 인지
  - 10초 이상: 사용자는 다른 일을 하기 시작

이 시스템에 적용하면:
  - 포털 초기 로딩: 3초 이내 (First Contentful Paint)
  - 앱 전환: 1초 이내 (프리페칭 시 0.5초)
  - API 응답: 200ms 이내 (P95)
  - 마스터 데이터 조회: 100ms 이내 (캐시 히트 시)
  - 검색 결과 표시: 500ms 이내
```

**Fowler:** 이 숫자를 **Performance Budget**으로 관리해야 합니다. 각 Remote App의 번들 크기, API 응답 시간에 상한을 설정하고, CI에서 자동 체크합니다.

```
Performance Budget:

Frontend:
  - Remote App 초기 번들: 200KB 이하 (gzip 기준)
  - 공유 디펜던시 제외 (Shell에서 로딩)
  - Lazy Loading 활용: 초기 로딩에 필요한 코드만 포함

Backend:
  - API P95 응답 시간: 200ms
  - API P99 응답 시간: 500ms
  - 데이터베이스 쿼리: 50ms 이내
  - 외부 서비스 호출: Circuit Breaker 타임아웃 3초
```

### 성능 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **PERF-1** | Performance Budget 정의 | Frontend 번들 크기 상한(Remote App별 200KB gzip). API 응답 시간 상한(P95 200ms, P99 500ms). 포털 초기 로딩 상한(FCP 3초, LCP 5초). CI에서 자동 체크: 상한 초과 시 PR 차단. |
| **PERF-2** | 캐싱 전략 | CDN: 정적 리소스(JS, CSS, 이미지) Edge 캐싱. API 캐싱: Redis, 마스터 데이터 TTL 5분, 권한 정보 TTL 1분. 브라우저 캐싱: Service Worker로 Shell App 오프라인 지원. 캐시 무효화: Event Bus로 변경 시 즉시 무효화. |
| **PERF-3** | 부하 테스트 계획 | 도구: k6 또는 Artillery. 시나리오: 동시 사용자 1000명, 포털 로그인 + 앱 전환 + 검색. 목표: P95 응답 500ms 이내, 에러율 0.1% 이하. 정기 실행: 주 1회 자동 실행 + 릴리즈 전 필수 실행. |

---

## 4-5. 마이그레이션 전략 (MIG-1~3)

**Beck:** 기존 프로젝트를 포털로 마이그레이션하는 전략도 정해야 합니다. 이건 기술보다 **조직적 문제**에 가깝습니다.

**Fowler:** 마이그레이션은 **Strangler Fig Pattern**이 답입니다. 기존 시스템을 한 번에 교체하는 게 아니라, 새 포털을 기존 시스템 앞에 놓고 점진적으로 트래픽을 옮기는 것입니다.

```
Strangler Fig 전략:

Phase 1: 공존
  - 기존 시스템과 포털이 동시에 운영
  - 포털은 기존 시스템으로의 링크도 제공
  - 사용자는 두 곳 모두 접근 가능

Phase 2: 전환
  - 새 기능은 포털에서만 제공
  - 기존 시스템의 사용 빈도 모니터링
  - 사용 빈도 낮은 기능부터 포털로 이전

Phase 3: 완료
  - 기존 시스템 접근을 포털로 리다이렉트
  - 기존 시스템 READ-ONLY 모드로 전환
  - 충분한 기간 후 기존 시스템 폐기
```

**Uncle Bob:** 각 프로젝트에 **마이그레이션 가이드**를 제공해야 합니다. "기존 독립형 앱을 포털 Remote App으로 변환하는 방법" 단계별 문서가 필요합니다.

```
프로젝트 마이그레이션 체크리스트:

□ 1. SDK 의존성 추가 (@portal/react-adapter, @portal/auth-sdk 등)
□ 2. 기존 인증 로직을 @portal/auth-sdk로 교체
□ 3. 기존 마스터 데이터 직접 조회를 @portal/master-sdk로 교체
□ 4. Module Federation 설정 추가 (webpack.config.js)
□ 5. Remote Entry 파일 작성 (createReactRemoteApp)
□ 6. manifest.json 작성 (라우팅, 권한, 메타데이터)
□ 7. Design Token 적용 (CSS 변수 import)
□ 8. 독립 실행 모드 유지 (포털 없이도 단독 실행 가능)
□ 9. Contract Test 작성 (Shell과의 계약 검증)
□ 10. 스테이징 환경에서 통합 테스트
```

### 마이그레이션 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **MIG-1** | Strangler Fig 전략 | 기존 시스템과 포털의 공존 기간 계획(최소 3개월). 트래픽 라우팅 전략(URL 기반 또는 Feature Flag 기반). 롤백 계획: 포털에 문제 시 기존 시스템으로 즉시 복귀 가능. 데이터 동기화: 전환 기간 동안 양 시스템 간 데이터 일관성 유지. |
| **MIG-2** | 프로젝트별 마이그레이션 가이드 | 10단계 체크리스트 제공. 프레임워크별 예시 코드(React, Vue, Angular). 예상 소요 시간: 프로젝트 규모별 추정(소규모 2주, 중규모 4주, 대규모 8주). 마이그레이션 지원: 페어 프로그래밍 세션 제공. |
| **MIG-3** | 독립 실행 모드 유지 | 마이그레이션 후에도 각 프로젝트 앱은 독립적으로 실행 가능해야 함. Dual-mode 설정: 포털 내부에서도, 단독으로도 동작. 독립 실행 시 인증은 자체 로그인 페이지 사용. 이유: 포털 장애 시에도 각 프로젝트는 독립적으로 서비스 가능. |

---

## 4-6. 다국어(i18n) / 멀티 테넌시 (EXT-1~3)

**Beck:** 마지막으로, 다국어 지원과 멀티 테넌시입니다. 현재 필요하지 않더라도, 아키텍처 수준에서 확장 가능하도록 고려해야 합니다.

**Nielsen:** 다국어는 "나중에 추가"하기가 정말 어렵습니다. 처음부터 **i18n 구조를 잡아놓고**, 한국어만 넣어 시작하는 게 맞습니다.

```
i18n 아키텍처:

1. 메시지 관리:
   - Shell App: @portal/i18n (공통 메시지)
   - Remote App: 각자의 i18n (프로젝트 메시지)
   - 마스터 데이터: DB에 다국어 컬럼 (name_ko, name_en, ...)

2. 로케일 전파:
   - Shell App에서 로케일 설정
   - AppContext.i18n.locale로 Remote App에 전달
   - Remote App은 전달받은 로케일로 렌더링

3. 날짜/숫자 포맷:
   - Intl API 사용 (브라우저 내장)
   - 로케일별 포맷 자동 적용
```

**Fowler:** 멀티 테넌시는 현재 필요하지 않더라도, 데이터 격리 수준만 정해놓으면 됩니다.

```
멀티 테넌시 격리 수준 (나중에 필요할 때를 위해):

Level 1: 공유 DB + Row-Level Security (tenant_id 컬럼)
  - 비용 최저, 격리 최약

Level 2: 스키마 분리 (tenant별 DB 스키마)
  - 중간 비용, 중간 격리

Level 3: DB 인스턴스 분리 (tenant별 DB)
  - 비용 최고, 격리 최강

현재 권장: Level 1 준비 (tenant_id 컬럼 예약)
  - 모든 주요 테이블에 tenant_id 컬럼을 두되, 현재는 단일 값
  - 나중에 멀티 테넌시가 필요하면 Row-Level Security만 활성화
```

### i18n/멀티 테넌시 검토 항목

| ID | 항목 | 상세 |
|----|------|------|
| **EXT-1** | i18n 아키텍처 | Shell App에서 로케일 관리, AppContext로 Remote App에 전달. 메시지 키 네이밍 규칙: `<domain>.<module>.<key>` (예: `auth.login.button_label`). 마스터 데이터 다국어: 다국어 컬럼 방식(name_ko, name_en) 또는 별도 번역 테이블. 날짜/숫자/통화 포맷: Intl API 기반 로케일별 자동 적용. |
| **EXT-2** | 멀티 테넌시 준비 | 주요 테이블에 `tenant_id` 컬럼 예약(현재는 단일 값). API에 `X-Tenant-ID` 헤더 규약 정의(현재는 무시). 격리 수준 로드맵: 현재 Level 1(공유 DB) → 필요 시 Level 2/3 전환. 테넌트별 설정: Feature Flags, Design Token 오버라이드 지원 구조. |
| **EXT-3** | 확장성 설계 원칙 | 수평 확장(Horizontal Scaling): Stateless 서비스 설계, 세션은 외부 저장소(Redis). 수직 확장 제한: 단일 서비스의 리소스 상한 정의. 데이터 파티셔닝: 프로젝트별/기간별 파티셔닝 전략. Auto-scaling: CPU/메모리 기반 + 커스텀 메트릭(동시 사용자 수) 기반. |

---

**Beck (사회자):** Session 4에서 **19개의 보완 검토 항목**(OPS 4개, DX 3개, GOV 3개, PERF 3개, MIG 3개, EXT 3개)을 추가했습니다. Session 1의 27개와 합치면 총 **46개의 검토 항목**입니다.

---

# Session 5: 전문가 클로징

**Beck (사회자):** 마지막 세션입니다. 각 전문가가 이번 브레인스토밍에서 가장 중요하다고 생각하는 메시지를 **한 문장**으로 남겨주세요.

---

## 5-1. 전문가 클로징 메시지

| # | 전문가 | 클로징 메시지 |
|---|--------|---------------|
| 1 | **Andrew Ng** | "Phase 1부터 데이터 파이프라인을 깔아두세요. AI는 데이터가 쌓인 후에 빛을 발합니다." |
| 2 | **Andrej Karpathy** | "자연어 검색, 이상 탐지, 문서 자동 생성 - 이 세 가지가 이 포털을 경쟁 시스템과 차별화할 것입니다." |
| 3 | **Martin Kleppmann** | "Canonical Data Model 없이 마스터 데이터 통합은 환상입니다. 거버넌스가 기술보다 먼저입니다." |
| 4 | **Martin Fowler** | "Integration at the edge, autonomy at the core. 세 개의 경계면(Frontend, Auth, Data) 외에는 결합하지 마세요." |
| 5 | **Uncle Bob** | "Contract-First, SDK 제공, 자동화 테스트. 이 세 가지가 없으면 6개 프로젝트의 통합은 카오스가 됩니다." |
| 6 | **Kent Beck** | "Tracer Bullet으로 시작하세요. 플랫폼을 먼저 만들지 말고, 프로젝트를 올리면서 플랫폼을 키우세요." |
| 7 | **Jakob Nielsen** | "이 포털은 사용자의 매일의 진입점입니다. Design Token과 통합 검색이 사용자 경험의 핵심입니다." |
| 8 | **Bruce Schneier** | "Zero Trust와 Audit Trail은 선택이 아닙니다. 모든 요청을 검증하고, 모든 접근을 기록하세요." |

---

## 5-2. 전체 46개 검토 항목 요약표

| # | ID | 영역 | 항목 | 우선 Phase |
|---|-----|------|------|-----------|
| 1 | AI-1 | AI | 자연어 검색 파이프라인 | Phase 3 |
| 2 | AI-2 | AI | 이상 접근 탐지 모델 | Phase 3 |
| 3 | AI-3 | AI | LLM 통합 아키텍처 | Phase 3 |
| 4 | AI-4 | AI | 데이터 파이프라인 선행 설계 | Phase 1 |
| 5 | AI-5 | AI | AI 추천 시스템 | Phase 4 |
| 6 | AI-6 | AI | AI 윤리 및 거버넌스 | Phase 3 |
| 7 | DB-1 | DB | 마스터 데이터 표준 모델 | Phase 0 |
| 8 | DB-2 | DB | 데이터 동기화 전략 | Phase 1 |
| 9 | DB-3 | DB | 권한 데이터 모델 | Phase 0 |
| 10 | DB-4 | DB | 감사 로그 저장소 | Phase 1 |
| 11 | ARCH-1 | 아키텍처 | Micro Frontend 아키텍처 | Phase 0 |
| 12 | ARCH-2 | 아키텍처 | 인증/인가 아키텍처 | Phase 0 |
| 13 | ARCH-3 | 아키텍처 | API 아키텍처 | Phase 0 |
| 14 | ARCH-4 | 아키텍처 | 장애 격리 및 복원 | Phase 1 |
| 15 | SW-1 | SW | Contract-First 개발 프로세스 | Phase 0 |
| 16 | SW-2 | SW | 공통 SDK 설계 | Phase 0 |
| 17 | SW-3 | SW | 테스트 전략 | Phase 1 |
| 18 | SW-4 | SW | 코드 품질 가드레일 | Phase 0 |
| 19 | PM-1 | PM | Tracer Bullet 대상 선정 | Phase 0 |
| 20 | PM-2 | PM | 단계별 롤아웃 계획 | Phase 0 |
| 21 | PM-3 | PM | 이해관계자 관리 | Phase 0 |
| 22 | PM-4 | PM | 위험 관리 | Phase 0 |
| 23 | UX-1 | UX | Design Token 표준 | Phase 0 |
| 24 | UX-2 | UX | 통합 검색 UX | Phase 2 |
| 25 | SEC-1 | 보안 | Zero Trust 구현 | Phase 1 |
| 26 | SEC-2 | 보안 | 감사 로그 체계 | Phase 1 |
| 27 | SEC-3 | 보안 | 취약점 관리 프로세스 | Phase 1 |
| 28 | OPS-1 | DevOps | CI/CD 파이프라인 설계 | Phase 0 |
| 29 | OPS-2 | DevOps | 구조화 로그 표준 | Phase 1 |
| 30 | OPS-3 | DevOps | 분산 추적(Distributed Tracing) | Phase 1 |
| 31 | OPS-4 | DevOps | 알림 및 온콜 체계 | Phase 1 |
| 32 | DX-1 | DX | CLI 도구 (portal-cli) | Phase 1 |
| 33 | DX-2 | DX | 개발자 문서 포털 | Phase 1 |
| 34 | DX-3 | DX | 로컬 개발 환경 | Phase 0 |
| 35 | GOV-1 | 거버넌스 | 데이터 리니지 추적 | Phase 2 |
| 36 | GOV-2 | 거버넌스 | 데이터 분류 및 보호 | Phase 1 |
| 37 | GOV-3 | 거버넌스 | 데이터 품질 관리 | Phase 2 |
| 38 | PERF-1 | 성능 | Performance Budget 정의 | Phase 0 |
| 39 | PERF-2 | 성능 | 캐싱 전략 | Phase 1 |
| 40 | PERF-3 | 성능 | 부하 테스트 계획 | Phase 2 |
| 41 | MIG-1 | 마이그레이션 | Strangler Fig 전략 | Phase 1 |
| 42 | MIG-2 | 마이그레이션 | 프로젝트별 마이그레이션 가이드 | Phase 1 |
| 43 | MIG-3 | 마이그레이션 | 독립 실행 모드 유지 | Phase 1 |
| 44 | EXT-1 | 확장 | i18n 아키텍처 | Phase 1 |
| 45 | EXT-2 | 확장 | 멀티 테넌시 준비 | Phase 2 |
| 46 | EXT-3 | 확장 | 확장성 설계 원칙 | Phase 1 |

---

**Beck (사회자):** 이것으로 Session 1~5의 브레인스토밍을 마칩니다. 총 46개의 검토 항목이 도출되었고, Phase 0~4 로드맵이 확정되었습니다. 이 내용을 기반으로 PRD와 TRD를 작성하면, AI가 구현 가능한 수준의 구체적인 설계 문서가 완성될 것입니다.

> **다음 단계:** Part 2(포털 설계 심화 토의)에서 Micro Frontend 통합 방식, Shell App 설계, Design System 전략을 구체화합니다.
