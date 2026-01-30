# Part 5: 통합 및 공통 설계 브레인스토밍 요약

## 개요
- **주제**: 전체 시스템 통합, API Gateway/BFF, 공통 SDK, CI/CD, 관찰가능성, 테스트, DX
- **참여 전문가**: Andrew Ng, Andrej Karpathy (AI), Martin Kleppmann (DB), Martin Fowler (아키텍처), Robert C. Martin (SW), Kent Beck (PM), Jakob Nielsen, Bruce Schneier (UX/보안)

---

## 1. 전체 시스템 아키텍처 (5계층)

- **Client Layer**: Portal Shell + Remote Apps, SDK 포함
- **Edge Layer**: CDN + API Gateway (Rate Limiting, JWT 검증, CORS)
- **BFF Layer (DMZ)**: API Aggregation, Response Transform, WebSocket Proxy
- **Service Layer (Private)**: Auth, MDM Hub, App Registry, Notification, Audit
- **Data Layer**: PostgreSQL, Redis, Kafka, Elasticsearch
- **Cross-Cutting**: OpenTelemetry, HashiCorp Vault
- **네트워크 세그먼트**: Public / DMZ / Private 3단계 보안 심층 방어

---

## 2. API Gateway / BFF 설계

- **Gateway**: Kong 채택 (플러그인 생태계, 검증된 솔루션)
- **Rate Limiting**: 인증 10회/분, 일반 100회/분, Lookup 1000회/분, 관리자 200회/분
- **BFF**: REST + BFF 우선 (GraphQL 추후), Promise.allSettled로 부분 실패 허용
- **통신**: REST(동기) + Kafka(비동기) + WebSocket(실시간)

---

## 3. 공통 SDK 명세

### Auth SDK (`@portal/auth-sdk`)
- SSO 로그인/로그아웃, 동기 권한 체크, 자동 토큰 갱신 (만료 5분 전)
- TypeScript, Java, Python, Go 지원

### Master SDK (`@portal/master-sdk`)
- 코드 조회(캐시 적용), 유효성 검증, 변경 구독
- TTL 기반 캐시 + 이벤트 기반 무효화, preload() 지원

### Portal SDK (`@portal/portal-sdk`)
- 앱 등록(AppManifest), Shell 통신, 이벤트 발행/구독, UI 통합(알림, 모달)
- MountProps로 AuthSDK, MasterSDK, PortalSDK 주입

### 버전 관리
- SemVer, 2개 메이저 버전 하위 호환 보장

---

## 4. CI/CD 파이프라인

- **컴포넌트별 독립 파이프라인** (Shell, Remote App, SDK, Backend)
- **필수 게이트**: Remote App은 Portal Integration Test 필수
- **Contract Testing**: Pact (Consumer-Driven)
- **보안 스캐닝 4단계**: SonarQube(SAST) + ZAP(DAST) + Snyk(의존성) + Trivy(이미지)
- **배포 전략**: Blue-Green(서비스) + Canary(프론트엔드), 롤백 5분 이내

---

## 5. 관찰가능성 (Logging, Metrics, Tracing)

### Logging
- 구조화된 JSON, PII 자동 마스킹, 30일 Hot / 90일 Warm / 1년 Cold

### Metrics
- Golden Signals (p95 < 200ms, 5xx < 0.1%) + 비즈니스 메트릭
- 에스컬레이션: warning(Slack 15분) -> critical(PagerDuty 5분)

### Tracing
- OpenTelemetry + W3C Trace Context
- 샘플링: 에러 100%, 정상 10%(프로덕션), Staging/Dev 100%

---

## 6. 테스트 전략 (피라미드 구조)

- **Unit**: 80%+ 커버리지, Jest, 5분 이내
- **Contract**: Pact, PR 머지 전 필수
- **Integration**: Docker Compose/Testcontainers
- **Portal Integration**: Remote App 배포 전 필수 게이트
- **E2E**: Playwright (4대 핵심 시나리오)
- **Performance**: k6 (p95 < 200ms, 에러율 < 0.1%)
- **Security**: OWASP ZAP (High 이상 시 배포 차단)
- **권한 테스트**: 매트릭스 기반 자동화 (역할 x 리소스 x 액션)

---

## 7. 개발자 경험(DX) 설계

- **portal-cli**: create-app, dev, test, deploy, register
- **로컬 개발**: Docker Compose로 전체 스택 (30초 시작)
- **문서**: 7개 카테고리 (Getting Started, Architecture, SDK Ref, Cookbook, API Ref, Troubleshooting, ADR)
- **Developer Portal**: Self-service 대시보드 (API Key, 앱 등록, Swagger UI, 로그 뷰어, Sandbox)

---

## 핵심 결정 사항 (34개)

| 영역 | 주요 결정 |
|------|----------|
| 통신 | REST + Kafka + WebSocket, 3단계 네트워크, Vault |
| Gateway/BFF | Kong, 차등 Rate Limiting, REST+BFF, 부분 실패 허용 |
| SDK | 동기 권한/자동 갱신, TTL+이벤트 캐시, 이벤트 기반 통신, SemVer |
| CI/CD | 독립 파이프라인, Portal 통합 테스트 게이트, Pact, 4단계 보안, Blue-Green/Canary |
| 관찰가능성 | JSON 로그+PII 마스킹, Golden Signals, OpenTelemetry, 차등 샘플링 |
| 테스트 | 피라미드(Unit 80%+), Playwright E2E, 권한 매트릭스, k6 성능 |
| DX | portal-cli, Docker Compose, 7개 문서, Developer Portal |
