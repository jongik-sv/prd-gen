# Part 3: 권한 관리(Authorization) 심화 토의 요약

## 개요
- **주제**: 권한 모델 상세 설계, 인증 흐름, DB 스키마, API 설계, OPA 연동, UI/UX, 감사 로그
- **참여 전문가**: Andrew Ng, Andrej Karpathy (AI), Martin Kleppmann (DB), Martin Fowler (아키텍처), Robert C. Martin (SW), Kent Beck (PM), Jakob Nielsen, Bruce Schneier (UX/보안)

---

## 1. 권한 모델 상세 설계 (RBAC + Project Scope)

### 역할 2축 구조
- **System Role (전역)**: SUPER_ADMIN, SYSTEM_ADMIN, SYSTEM_AUDITOR
- **Project Role (프로젝트 범위)**: PROJECT_ADMIN, PROJECT_MEMBER, PROJECT_VIEWER
- Permission 형식: `resource:action`, 프로젝트 범위 시 `resource:action@project`

### Role Template 운영
- 시스템 기본 템플릿 제공, 프로젝트 관리자가 clone하여 커스터마이징
- Role Hierarchy 지원: `parentRoleId`로 역할 상속

### ABAC 확장
- `PolicyCondition` 인터페이스 Phase 1에서 정의만, Phase 2에서 OPA 연동 시 활성화

---

## 2. 인증 흐름 설계

### OAuth2 Authorization Code + PKCE
- Phase 1: PKCE 기반 SPA 직접 통신, Phase 2: BFF 패턴 전환 검토

### Token 관리
- **Access Token**: JWT, 15분, 메모리 저장
- **Refresh Token**: Opaque, 7일, HttpOnly Secure Cookie, 서버측 저장
- Refresh Token Rotation + 탈취 감지 메커니즘

### Session 관리
- 절대 타임아웃 8시간, 유휴 30분, 동시 최대 3세션
- 초과 시 사용자에게 디바이스 정보 표시 + 선택권 제공

---

## 3. 권한 DB 스키마 (PostgreSQL 13개 테이블)

- Core: users, roles, permissions, role_permissions
- Project: projects, project_members, project_member_roles
- Delegation/Approval: permission_delegations, approval_requests
- Audit/Session: audit_logs, refresh_tokens, sessions, resources, user_system_roles
- 감사 로그 비정규화 + 불변성 트리거, Refresh Token은 해시값만 DB 저장

---

## 4. 권한 체크 API 설계

### 6개 API 그룹
1. 인증 API (login, refresh, logout, me)
2. 권한 체크 API (단일, 일괄 batch, 내 권한)
3. 역할 관리 API (CRUD, templates, clone)
4. 사용자-역할 할당 API
5. 위임 API
6. 승인 워크플로우 API

### 에러 코드 체계
- AUTH_001~005, PERM_001~004, PROJ_001~003, VAL_001, SYS_001

---

## 5. Policy Engine (OPA) 연동

- **Hybrid 패턴**: API Gateway(기본 인증) + 서비스별 OPA Sidecar(세밀한 인가)
- **장애 Fallback**: read=DB RBAC 체크, write=기본 거부
- **캐시 TTL**: 결정 60초, 번들 30초, 데이터 5분 + 이벤트 무효화

---

## 6. 권한 관리 UI/UX (8개 핵심 화면)

1. 역할 관리 / 2. Permission Matrix / 3. 내 권한 조회 / 4. 역할 비교(Diff)
5. 권한 요청 워크플로우 / 6. 권한 위임 / 7. CSV Import / 8. 대시보드

### 이상 활동 탐지 (6가지 규칙)
- 다수 권한 거부, 비정상 시간대 로그인, 비정상 IP, 비활성 계정, 동시 다른 지역 로그인, 대량 데이터 조회
- Phase 1: 규칙 기반, Phase 2: ML 기반 확장

---

## 7. 감사 로그 설계

- **3대 원칙**: 불변성, 완전성, 검색성
- **3-tier 저장**: Hot(PostgreSQL, 6개월) + Warm(ES, 1년) + Cold(S3, 3년+)
- **무결성**: 시간 블록 해시 검증
- **SIEM 통합**: Syslog(CEF) + Webhook + Kafka

---

## 핵심 결정 사항 (14개)

| # | 항목 | 결정 |
|---|------|------|
| 1 | 권한 모델 | System Role + Project Role 2축, `resource:action@project` |
| 2 | Role Template | 시스템 템플릿 clone + 커스터마이징, 상속 지원 |
| 3 | ABAC 확장 | Phase 1 정의, Phase 2 OPA 활성화 |
| 4 | 인증 흐름 | OAuth2 + PKCE (Phase 1), BFF (Phase 2) |
| 5 | Token | JWT 15분 + Opaque 7일, Rotation + 탈취 감지 |
| 6 | Session | 절대 8h, 유휴 30m, 동시 3세션 |
| 7 | DB 스키마 | PostgreSQL 13개 테이블, 불변 트리거 |
| 8 | API | 6개 그룹, batch 권한 체크, 체계적 에러 코드 |
| 9 | OPA | Hybrid 패턴, 장애 시 read=DB/write=거부 |
| 10 | 캐시 | 결정 60초, 번들 30초, 데이터 5분 |
| 11 | UI/UX | 8개 핵심 화면 |
| 12 | 감사 로그 | DB 분리, 3-tier, 시간 블록 해시, SIEM |
| 13 | 에러 코드 | AUTH/PERM/PROJ/VAL/SYS 체계 |
| 14 | 이상 탐지 | Phase 1 규칙(6패턴), Phase 2 ML |
