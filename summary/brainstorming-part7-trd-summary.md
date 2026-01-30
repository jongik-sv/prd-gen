# Part 7: TRD(기술 참조 문서) 브레인스토밍 종합 요약

## 개요
- **주제**: 시스템 아키텍처, 디렉토리 구조, TypeScript 타입, API 설계, DB 스키마(DDL), 시퀀스 다이어그램, 환경 설정, 테스트 시나리오
- **참여 전문가**: Andrew Ng, Andrej Karpathy, Martin Kleppmann, Martin Fowler, Robert C. Martin, Kent Beck, Jakob Nielsen, Bruce Schneier, Donald Knuth

---

## 1. 시스템 아키텍처 (7.1)

### 5개 레이어
- Client -> Edge(CDN, Kong Gateway) -> Application(BFF, Auth, MDM, Notification) -> Data(PG, Redis, ES, Kafka) -> Cross-cutting(OTel, Vault)

### 통신 패턴
- 동기: REST API/gRPC, 비동기: Kafka, 실시간: WebSocket

### 장애 대응
- 서비스별 독립 Circuit Breaker (CLOSED -> OPEN -> HALF-OPEN)
- 4개 독립 장애 도메인: PG(Replica), Redis(로컬 폴백), ES(비활성화), Kafka(동기 폴백)

### 보안 영역 (4개)
- Public(CDN/LB) -> DMZ(Gateway/WAF) -> Private(Services) -> Data(DB/Cache/MQ)
- 영역 간 mTLS, Vault 인증서 24시간 자동 갱신

### 배포 단위: 11개 독립 배포 단위

---

## 2. 디렉토리 구조 (7.2)

### 모노레포 최상위
- `apps/` (Shell, Auth Admin, Master Admin)
- `packages/` (design-tokens, ui, auth-sdk, master-sdk, portal-sdk, shared-types)
- `services/` (auth, mdm, portal-bff, notification)
- `infra/` (Docker, K8s, Terraform)

### 핵심 원칙
- 단방향 의존: apps/services -> packages (앱 간 직접 import 금지)
- Clean Architecture, Feature Co-location
- Turborepo + pnpm workspace, 해시 기반 캐싱 + Remote Cache

---

## 3. TypeScript 타입 정의 (7.3)

- **공통**: ApiResponse\<T\>, PaginatedResponse\<T\>, FilterOption (Zod 런타임 검증)
- **Auth**: User, Role, Permission, TokenPayload, Delegation, ApprovalRequest
- **Master**: CodeGroup, MasterCode, ChangeRequest, BulkImportJob (낙관적 잠금 version)
- **Portal**: AppManifest, ShellState, Notification
- **Event**: DomainEvent\<T\> (correlationId 분산 추적)
- enum 대신 union type (트리 쉐이킹 유리)

---

## 4. API 엔드포인트 설계 (7.4)

- **Auth API**: SSO 로그인, OIDC 콜백, 토큰 갱신, 로그아웃
- **Permission API**: 단일/배치 권한 체크, 역할 CRUD, 위임, 승인
- **Master API**: 코드 그룹/코드 CRUD, 캐시 조회(ETag), 벌크, 검색
- **Portal API**: 앱 목록, 대시보드 집약, 통합 검색, 알림
- **에러 코드**: 19개 (AUTH/VALIDATION/RESOURCE/MASTER/PORTAL/SYSTEM)
- **Rate Limiting**: Auth 10/min, Read 100/min, Write 30/min, Bulk 5/min
- **API 버전**: URL 경로 `/api/v1/`, 이전 버전 최소 6개월 병행

---

## 5. DB 스키마 DDL (7.5)

### Auth DB (15개 테이블)
- users, roles, resources, permissions, role_permissions
- user_system_roles, projects, project_members, project_member_roles
- permission_delegations, approval_requests, approval_steps
- audit_logs (월별 파티션, 불변 트리거), refresh_tokens, sessions

### Master DB (10개 테이블)
- code_groups, code_group_names, master_codes, master_code_names
- master_code_attributes, code_versions, code_dependencies, code_mappings
- bulk_import_jobs, outbox_events (Transactional Outbox)

### 주요 설계
- 월별 RANGE 파티션 (pg_partman), 복합 인덱스 + partial 인덱스
- audit_logs UPDATE/DELETE 차단 트리거

---

## 6. 시퀀스 다이어그램 (7.6, 6가지 핵심 흐름)

1. **SSO 로그인**: OIDC Authorization Code Flow (Keycloak)
2. **권한 체크**: Gateway(인증) + OPA Sidecar(인가) 이중 구조
3. **코드 조회**: SDK -> L1(메모리) -> L2(Redis) -> API -> DB
4. **코드 변경 승인**: PENDING -> 승인 -> 적용 + Outbox -> Kafka -> 캐시 갱신
5. **MFE 로딩**: Registry -> 권한 체크 -> CDN remoteEntry.js -> mount()
6. **Token 갱신**: 401 인터셉트 -> Queue -> 단일 refresh -> 원본/대기 요청 재시도

---

## 7. 환경 설정 (7.7)

### 기술 스택
- Runtime: Node.js 20.11.x, TypeScript 5.4, pnpm 8.15
- Frontend: React 18.3, Vite 5.x, TailwindCSS 3.4, Zustand 4.x
- Backend: NestJS 10, TypeORM 0.3, KafkaJS 2.2
- Infra: Docker 25, K8s 1.29, Terraform 1.7
- 관측성: OpenTelemetry -> Grafana + Loki + Tempo + Prometheus

### Docker Compose (10개 서비스)
- PostgreSQL 16, Redis 7.2, Kafka(KRaft), Kafka UI, ES 8.12, Keycloak 24, Vault 1.15, Grafana, Loki, OTel Collector
- 모든 서비스 healthcheck + depends_on 기동 순서 제어

---

## 8. 테스트 시나리오 (7.8)

### 커버리지 목표
| 레벨 | 도구 | 목표 |
|------|------|------|
| Unit | Vitest | 80% line coverage |
| Integration | Testcontainers | 70% branch coverage |
| Contract | Pact 12 | 핵심 API 100% |
| E2E | Playwright | Critical Path 100% |
| Performance | k6 | P95 < 200ms, Error < 0.1% |

### E2E 4대 시나리오
1. SSO 로그인 -> 대시보드 -> 프로젝트 접근
2. 역할 생성 -> 권한 할당 -> 사용자 배정 -> 검증
3. 코드 생성 요청 -> 승인 -> 조회 확인
4. Excel 업로드 -> 검증 -> 승인 -> 적용

### 권한 매트릭스
- 8개 역할 x 12개 액션 = 96개 조합 테스트

---

## 핵심 결정 사항 총괄 (48개)

| 영역 | 결정 수 | 주요 항목 |
|------|---------|----------|
| 아키텍처 | 7 | MFE + 모듈러 모놀리스, Kong, Keycloak+JWT+OPA, L1/L2 캐시 |
| API 설계 | 5 | URL 경로 버전, Envelope 응답, 차등 Rate Limit, Cursor 페이지네이션 |
| 프로젝트 구조 | 4 | Turborepo+pnpm, 4영역 디렉토리, strict TS, 공유 패키지 |
| 코딩 컨벤션 | 3 | PascalCase/camelCase, ESLint+Prettier, Conventional Commits |
| DB 스키마 | 7 | PG 16, Auth 15테이블, Master 10테이블, 월별 파티션, Outbox |
| 시퀀스 | 6 | SSO, 권한 체크, 코드 조회, 변경 승인, MFE 로딩, Token 갱신 |
| 환경 설정 | 8 | Node 20, React 18, NestJS 10, Docker 25, K8s 1.29, Vault, OTel |
| 테스트 | 8 | Vitest 80%, Pact, Playwright, k6, 96개 권한 조합, MSW |
