# Part 3: 권한 관리(Authorization) 심화 토의 - 대화형 토론 기록

## 토의 참석자
| 분야 | 인물 |
|------|------|
| AI | Andrew Ng + Andrej Karpathy |
| DB | Martin Kleppmann |
| 아키텍처 | Martin Fowler |
| SW | Robert C. Martin (Uncle Bob) |
| PM | Kent Beck (사회자) |
| UX/보안 | Jakob Nielsen + Bruce Schneier |

## 이전 세션(1-2) 합의 사항 요약
- RBAC + Project Scope 기반, Policy Engine(OPA) 도입
- Central Auth Server (Keycloak 또는 자체), OAuth 2.0 + OIDC
- Token: Access Token (JWT 15-30분), Refresh Token (Opaque, 서버측), ID Token
- Zero Trust: 각 서비스 자체 Token 검증
- Audit Trail: 불변 append-only 로그

---

## 토의 3.1: 권한 모델 상세 설계 (RBAC + Project Scope)

**Beck (사회자):** 자, 오늘은 권한 관리를 깊이 파고듭니다. 세션 1-2에서 RBAC + Project Scope + OPA라는 큰 그림에 합의했으니, 이제 **구체적인 엔티티와 관계**를 정의합시다. Uncle Bob, 핵심 엔티티부터 정리해주시죠.

---

**Uncle Bob:** 권한 모델의 핵심 엔티티는 5개입니다:

```
1. User       - 시스템 사용자
2. Role       - 권한 묶음 (역할)
3. Permission - 개별 권한 (resource:action)
4. Resource   - 보호 대상 자원
5. Project    - 권한의 범위(scope)
```

이 엔티티들의 관계를 먼저 명확히 해야 합니다. User는 두 가지 차원에서 Role을 가집니다. **System Role**과 **Project Role**입니다.

**Fowler:** 구체적으로 역할 계층을 정의합시다. 제가 보기에 System Role과 Project Role은 완전히 다른 축입니다:

```
System Role (시스템 전역):
  - SUPER_ADMIN    : 모든 권한, 시스템 설정 변경
  - SYSTEM_ADMIN   : 사용자 관리, 프로젝트 생성/삭제
  - SYSTEM_AUDITOR : 전체 감사 로그 조회 (읽기 전용)

Project Role (프로젝트 범위):
  - PROJECT_ADMIN  : 프로젝트 내 모든 권한, 멤버 관리
  - PROJECT_MEMBER : 프로젝트 내 일반 작업 수행
  - PROJECT_VIEWER : 프로젝트 내 읽기 전용
```

System Role은 프로젝트와 무관하게 시스템 전체에 적용됩니다. Project Role은 반드시 특정 프로젝트에 바인딩됩니다.

**Beck:** 한 사용자가 프로젝트 A에서는 ADMIN이고, 프로젝트 B에서는 VIEWER일 수 있는 거죠?

**Fowler:** 정확합니다. 그게 **Project Scope**의 핵심입니다. 같은 사용자라도 프로젝트마다 다른 역할을 가집니다.

**Uncle Bob:** Permission의 형식을 정합시다. 저는 `resource:action` 포맷을 제안합니다:

```
Permission 형식: "{resource}:{action}"

예시:
  user:read           - 사용자 정보 조회
  user:write          - 사용자 정보 수정
  user:delete         - 사용자 삭제
  master-code:read    - 기준정보 코드 조회
  master-code:write   - 기준정보 코드 수정
  master-code:approve - 기준정보 코드 승인
  audit-log:read      - 감사 로그 조회
  role:read           - 역할 조회
  role:write          - 역할 수정
  project:create      - 프로젝트 생성
  project:delete      - 프로젝트 삭제
  dashboard:read      - 대시보드 조회
  report:export       - 보고서 내보내기
```

**Schneier:** 여기에 **Project Scope**를 결합하면 `permission@project` 형태가 됩니다:

```
"master-code:write@project-a"  -> 프로젝트 A에서 기준정보 코드 수정 권한
"user:read@project-b"          -> 프로젝트 B에서 사용자 정보 조회 권한
"master-code:approve@project-a" -> 프로젝트 A에서 기준정보 코드 승인 권한
```

System Role의 Permission은 `@` 없이 전역 적용됩니다. `user:delete`는 시스템 전체에서 사용자 삭제 가능을 의미합니다.

**Ng:** 여기서 질문입니다. 프로젝트마다 필요한 Permission 세트가 다를 수 있지 않나요? 프로젝트 A는 `approval` 워크플로우가 있고, 프로젝트 B는 없다면?

**Uncle Bob:** 좋은 질문입니다. 그래서 **Role Template** 개념이 필요합니다. 시스템에서 기본 Role Template을 제공하고, 프로젝트 관리자가 이를 **복제(clone)**해서 프로젝트에 맞게 커스터마이징합니다.

```
Role Template 운영 방식:
1. 시스템이 기본 템플릿 제공: "기본 멤버", "기본 관리자", "기본 뷰어"
2. 프로젝트 관리자가 템플릿을 clone
3. clone된 역할에서 Permission을 추가/제거
4. 프로젝트 멤버에게 커스터마이징된 역할 할당
```

**Fowler:** 거기에 **역할 상속(Role Hierarchy)**도 추가합시다. PROJECT_ADMIN은 PROJECT_MEMBER의 모든 권한을 포함하고, PROJECT_MEMBER는 PROJECT_VIEWER의 모든 권한을 포함합니다. 이걸 `parentRoleId`로 표현합니다.

```
PROJECT_ADMIN
  └── PROJECT_MEMBER (상속)
        └── PROJECT_VIEWER (상속)
```

이렇게 하면 PROJECT_ADMIN에는 관리 전용 Permission만 직접 할당하면 됩니다. 나머지는 부모 역할에서 상속됩니다.

**Uncle Bob:** 자, 이걸 TypeScript 인터페이스로 정리합시다:

```typescript
// ===== 권한 모델 핵심 인터페이스 =====

type SystemRoleName = 'SUPER_ADMIN' | 'SYSTEM_ADMIN' | 'SYSTEM_AUDITOR';
type ProjectRoleName = 'PROJECT_ADMIN' | 'PROJECT_MEMBER' | 'PROJECT_VIEWER';

interface User {
  id: string;
  email: string;
  name: string;
  status: 'active' | 'inactive' | 'locked';
  systemRoles: SystemRole[];
  projectAssignments: ProjectAssignment[];
  createdAt: Date;
  updatedAt: Date;
  lastLoginAt?: Date;
}

interface SystemRole {
  id: string;
  name: SystemRoleName;
  description: string;
  permissions: Permission[];
}

interface ProjectAssignment {
  projectId: string;
  roles: ProjectRole[];
  startDate: Date;
  endDate?: Date;         // temporal permission: 만료일 지정 가능
  assignedBy: string;     // 할당한 관리자 ID
  assignedAt: Date;
}

interface ProjectRole {
  id: string;
  name: string;           // 커스터마이징 가능하므로 string
  baseTemplateName?: ProjectRoleName; // 원본 템플릿 참조
  description: string;
  permissions: Permission[];
  isTemplate: boolean;    // true면 시스템 기본 템플릿
  parentRoleId?: string;  // 역할 상속 (상위 역할)
  projectId?: string;     // template이면 null, instance면 프로젝트 ID
}

interface Role {
  id: string;
  name: string;
  scope: 'system' | 'project';
  permissions: Permission[];
  isTemplate: boolean;
  parentRoleId?: string;  // role hierarchy
  description: string;
  createdAt: Date;
  updatedAt: Date;
}

interface Permission {
  id: string;
  resource: string;       // e.g., "master-code", "user", "audit-log"
  action: string;         // e.g., "read", "write", "delete", "approve"
  description: string;
  conditions?: PolicyCondition[]; // ABAC extension point
}

interface PolicyCondition {
  attribute: string;      // e.g., "time", "ip", "department"
  operator: 'eq' | 'neq' | 'in' | 'not_in' | 'gt' | 'lt' | 'between';
  value: any;
}

interface Resource {
  id: string;
  name: string;           // e.g., "master-code", "user", "audit-log"
  description: string;
  actions: string[];      // e.g., ["read", "write", "delete", "approve"]
}

interface Project {
  id: string;
  name: string;
  code: string;           // 프로젝트 코드 (URL-safe)
  status: 'active' | 'archived' | 'suspended';
  createdAt: Date;
  updatedAt: Date;
}
```

**Karpathy:** ABAC extension point인 `PolicyCondition`이 보이는데, 이건 Phase 1에서 구현하나요?

**Fowler:** Phase 1에서는 **인터페이스만 정의**하고 실제 조건 평가는 비활성화합니다. Phase 2에서 OPA와 연동해서 시간 기반, IP 기반 조건을 추가합니다. 확장 포인트만 미리 열어두는 겁니다.

**Schneier:** 동의합니다. 하지만 `PolicyCondition`의 스키마는 지금 확정해야 합니다. 나중에 바꾸면 마이그레이션이 고통입니다.

> **합의:**
> 1. System Role(전역) / Project Role(프로젝트 범위) 2축 구조
> 2. Permission 형식: `resource:action`, Project Scope 적용 시 `resource:action@project`
> 3. Role Template으로 기본 역할 제공, 프로젝트별 clone + 커스터마이징
> 4. Role Hierarchy: parentRoleId로 역할 상속 지원
> 5. ABAC PolicyCondition 확장 포인트는 Phase 1에서 인터페이스만 정의

---

## 토의 3.2: 인증 흐름 설계 (SSO, OAuth2/OIDC, Token 생명주기)

**Beck (사회자):** 다음은 인증 흐름입니다. 세션 1에서 OAuth2/OIDC, JWT Access Token + Opaque Refresh Token으로 합의했는데, 구체적인 **시퀀스**를 그려봅시다. Schneier, 리드해주시죠.

---

**Schneier:** 전체 인증 흐름은 4가지입니다:

```
1. Login Flow         - 사용자 최초 로그인
2. Token Refresh Flow - Access Token 갱신
3. Logout Flow        - 로그아웃 (Single Logout)
4. Service-to-Service - 서비스 간 인증
```

먼저 Login Flow부터 그립시다.

### Login Flow 시퀀스

```
[사용자 브라우저]          [포털 Shell App]          [Auth Server]          [IdP (LDAP/AD)]
      |                        |                        |                        |
      |  1. 포털 접속           |                        |                        |
      |----------------------->|                        |                        |
      |                        |                        |                        |
      |  2. 미인증 상태 감지     |                        |                        |
      |  3. Auth Server로       |                        |                        |
      |     리다이렉트           |                        |                        |
      |<-----------------------|                        |                        |
      |                        |                        |                        |
      |  4. /authorize 요청 (PKCE)                       |                        |
      |  (response_type=code,                            |                        |
      |   client_id, redirect_uri,                       |                        |
      |   code_challenge, state, nonce)                  |                        |
      |------------------------------------------------->|                        |
      |                        |                        |                        |
      |  5. 로그인 페이지 반환   |                        |                        |
      |<-------------------------------------------------|                        |
      |                        |                        |                        |
      |  6. credentials 입력    |                        |                        |
      |------------------------------------------------->|                        |
      |                        |                        |                        |
      |                        |                        |  7. 자격증명 검증        |
      |                        |                        |----------------------->|
      |                        |                        |                        |
      |                        |                        |  8. 검증 결과 반환       |
      |                        |                        |<-----------------------|
      |                        |                        |                        |
      |  9. Authorization Code  |                        |                        |
      |     + state 반환        |                        |                        |
      |     (redirect_uri로)    |                        |                        |
      |<-------------------------------------------------|                        |
      |                        |                        |                        |
      |  10. code + state 전달  |                        |                        |
      |----------------------->|                        |                        |
      |                        |                        |                        |
      |                        |  11. POST /token        |                        |
      |                        |  (code, code_verifier,  |                        |
      |                        |   client_id,            |                        |
      |                        |   client_secret)        |                        |
      |                        |------------------------>|                        |
      |                        |                        |                        |
      |                        |  12. Token 발급         |                        |
      |                        |  (access_token,         |                        |
      |                        |   refresh_token,        |                        |
      |                        |   id_token)             |                        |
      |                        |<------------------------|                        |
      |                        |                        |                        |
      |  13. Access Token ->    |                        |                        |
      |      메모리 저장         |                        |                        |
      |      Refresh Token ->   |                        |                        |
      |      HttpOnly Cookie    |                        |                        |
      |  14. 포털 메인 렌더링    |                        |                        |
      |<-----------------------|                        |                        |
```

**Uncle Bob:** PKCE(Proof Key for Code Exchange)를 사용하는 게 중요합니다. SPA에서는 client_secret을 안전하게 보관할 수 없으므로, PKCE로 Authorization Code 가로채기를 방지합니다.

**Fowler:** 한 가지 추가합니다. 11단계에서 Shell App이 직접 Auth Server에 token 요청을 보내는 게 아니라, **Backend for Frontend(BFF)** 패턴을 사용하는 게 더 안전합니다. BFF가 client_secret을 보관하고, 브라우저는 BFF에게만 통신합니다.

**Schneier:** 맞습니다. BFF 패턴이면 보안이 훨씬 강화됩니다:

```
브라우저 -> BFF(서버) -> Auth Server
             |
             +--> client_secret은 BFF 서버에만 존재
             +--> Refresh Token도 BFF 서버에서 관리
             +--> 브라우저에는 BFF 세션 쿠키만 전달
```

하지만 BFF를 도입하면 아키텍처 복잡도가 올라갑니다. **Phase 1에서는 PKCE 기반 SPA 직접 통신**, Phase 2에서 BFF 전환도 옵션입니다.

**Beck:** Phase 1은 PKCE SPA, Phase 2에 BFF 검토로 갑시다. Token 관련 인터페이스를 정리하죠.

### Token 인터페이스

```typescript
// ===== Token 관련 인터페이스 =====

interface AuthTokens {
  accessToken: string;    // JWT, 15min TTL
  refreshToken: string;   // Opaque, 7day TTL, server-side stored
  idToken: string;        // JWT, user info (OIDC)
  expiresIn: number;      // seconds until access token expires
  tokenType: 'Bearer';
}

interface JWTPayload {
  // Standard Claims
  sub: string;            // user ID
  iss: string;            // issuer (Auth Server URL)
  aud: string;            // audience (client_id)
  exp: number;            // expiration timestamp
  iat: number;            // issued at timestamp
  nbf: number;            // not before timestamp
  jti: string;            // JWT ID (for revocation check)

  // Custom Claims
  email: string;
  name: string;
  roles: string[];        // system roles: ["SYSTEM_ADMIN"]
  projects: ProjectClaim[];
}

interface ProjectClaim {
  id: string;             // project ID
  code: string;           // project code
  roles: string[];        // project roles: ["PROJECT_ADMIN"]
}

// Service-to-Service Token (Client Credentials Grant)
interface ServiceToken {
  accessToken: string;    // JWT, 5min TTL (짧게)
  expiresIn: number;
  tokenType: 'Bearer';
  scope: string;          // e.g., "master-data:read audit:write"
}
```

### Token Refresh Flow

```
[브라우저]                [포털 BFF/Shell]           [Auth Server]
    |                        |                        |
    |  1. API 호출 시         |                        |
    |     Access Token 만료   |                        |
    |     감지 (401 또는       |                        |
    |     exp 시간 체크)       |                        |
    |----------------------->|                        |
    |                        |                        |
    |                        |  2. POST /auth/token/refresh
    |                        |  (refresh_token)        |
    |                        |------------------------>|
    |                        |                        |
    |                        |                        |  3. Refresh Token 유효성 검증
    |                        |                        |     - DB에서 토큰 존재 확인
    |                        |                        |     - 만료 여부 확인
    |                        |                        |     - Rotation: 기존 토큰 폐기,
    |                        |                        |       새 토큰 발급
    |                        |                        |
    |                        |  4. 새 Access Token     |
    |                        |     + 새 Refresh Token   |
    |                        |<------------------------|
    |                        |                        |
    |  5. 새 Token으로        |                        |
    |     원래 API 재요청      |                        |
    |<-----------------------|                        |
```

**Kleppmann:** Refresh Token Rotation이 중요합니다. 갱신할 때마다 새 Refresh Token을 발급하고, 이전 것은 폐기합니다. 만약 폐기된 Refresh Token으로 갱신 시도가 오면, **모든 세션을 무효화**합니다. 이건 탈취 감지 메커니즘입니다.

### Logout Flow (Single Logout)

```
[브라우저]                [포털 Shell]              [Auth Server]           [각 프로젝트 서비스]
    |                        |                        |                        |
    |  1. 로그아웃 클릭       |                        |                        |
    |----------------------->|                        |                        |
    |                        |                        |                        |
    |                        |  2. POST /auth/logout   |                        |
    |                        |  (refresh_token, id_token_hint)                  |
    |                        |------------------------>|                        |
    |                        |                        |                        |
    |                        |                        |  3. Refresh Token 폐기  |
    |                        |                        |  4. 세션 종료           |
    |                        |                        |  5. Back-channel Logout |
    |                        |                        |     알림 (각 서비스에)    |
    |                        |                        |----------------------->|
    |                        |                        |                        |
    |  6. 메모리 Access Token 제거                      |                        |
    |  7. HttpOnly Cookie 삭제                          |                        |
    |  8. 로그인 페이지 리다이렉트                        |                        |
    |<-----------------------|                        |                        |
```

### Session Management

```typescript
interface SessionConfig {
  absoluteTimeout: number;    // 8시간 (28800초) - 로그인 후 최대 유지 시간
  idleTimeout: number;        // 30분 (1800초) - 비활동 시 자동 로그아웃
  maxConcurrentSessions: number; // 3 - 동시 로그인 가능 세션 수
  sessionRenewalWindow: number;  // 5분 (300초) - 만료 전 갱신 가능 시간
}

// 세션 초과 시 처리 전략
type SessionExceedStrategy = 'deny_new' | 'terminate_oldest';
// deny_new: 새 로그인 거부, terminate_oldest: 가장 오래된 세션 종료
```

**Nielsen:** 세션 초과 시 **사용자에게 선택권**을 주는 게 좋습니다. "이미 3개 세션이 있습니다. 가장 오래된 세션을 종료하고 로그인하시겠습니까?" 이런 UI가 필요합니다.

**Schneier:** 보안 관점에서는 `terminate_oldest`가 더 안전합니다. 공격자가 탈취한 세션이 오래된 것일 가능성이 높으니까요. 하지만 Nielsen 말대로 UX를 위해 선택 화면을 보여주되, **어떤 디바이스에서 로그인 중인지** 정보를 함께 표시해야 합니다.

> **합의:**
> 1. OAuth2 Authorization Code + PKCE 기반 로그인 (Phase 1), BFF 패턴은 Phase 2 검토
> 2. Token 저장: Access Token(메모리), Refresh Token(HttpOnly Secure Cookie)
> 3. Refresh Token Rotation: 갱신 시 새 토큰 발급, 폐기된 토큰 사용 감지 시 전체 세션 무효화
> 4. Single Logout: Back-channel Logout으로 모든 서비스에 세션 종료 알림
> 5. Session: 절대 8시간, 유휴 30분, 동시 3세션, 초과 시 사용자 선택 + 디바이스 정보 표시

---

## 토의 3.3: 권한 DB 스키마 설계 (ERD)

**Beck (사회자):** 이제 데이터베이스로 갑시다. Kleppmann, 리드해주세요.

---

**Kleppmann:** 앞서 정의한 엔티티들을 PostgreSQL DDL로 구체화합니다. 핵심 테이블은 13개입니다:

```
Core:        users, roles, permissions, role_permissions
Project:     projects, project_members, project_member_roles
Delegation:  permission_delegations
Approval:    approval_requests
Audit:       audit_logs
Session:     refresh_tokens, sessions
Resource:    resources
```

### 전체 DDL

```sql
-- =============================================
-- 1. 사용자 테이블
-- =============================================
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'inactive', 'locked')),
    password_hash   VARCHAR(255),           -- 내부 인증 시 사용
    external_id     VARCHAR(255),           -- 외부 IdP 사용자 ID
    department      VARCHAR(100),
    phone           VARCHAR(50),
    last_login_at   TIMESTAMPTZ,
    login_fail_count INT NOT NULL DEFAULT 0,
    locked_until    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_external_id ON users(external_id);

-- =============================================
-- 2. 리소스 테이블
-- =============================================
CREATE TABLE resources (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(100) NOT NULL UNIQUE,   -- e.g., "master-code", "user"
    description TEXT,
    actions     TEXT[] NOT NULL,                 -- e.g., {"read","write","delete","approve"}
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================
-- 3. 권한 테이블
-- =============================================
CREATE TABLE permissions (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource    VARCHAR(100) NOT NULL,   -- e.g., "master-code"
    action      VARCHAR(50) NOT NULL,    -- e.g., "write"
    description TEXT,
    conditions  JSONB,                   -- ABAC 확장: PolicyCondition[]
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(resource, action)
);

CREATE INDEX idx_permissions_resource ON permissions(resource);
CREATE INDEX idx_permissions_resource_action ON permissions(resource, action);

-- =============================================
-- 4. 역할 테이블
-- =============================================
CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(100) NOT NULL,
    scope           VARCHAR(20) NOT NULL CHECK (scope IN ('system', 'project')),
    description     TEXT,
    is_template     BOOLEAN NOT NULL DEFAULT false,
    parent_role_id  UUID REFERENCES roles(id),
    project_id      UUID,                   -- project scope 역할 시 프로젝트 ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(name, scope, project_id)         -- 같은 범위+프로젝트 내 이름 유일
);

CREATE INDEX idx_roles_scope ON roles(scope);
CREATE INDEX idx_roles_project_id ON roles(project_id);
CREATE INDEX idx_roles_is_template ON roles(is_template);
CREATE INDEX idx_roles_parent_role_id ON roles(parent_role_id);

-- =============================================
-- 5. 역할-권한 매핑 (Many-to-Many)
-- =============================================
CREATE TABLE role_permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(role_id, permission_id)
);

CREATE INDEX idx_role_permissions_role_id ON role_permissions(role_id);
CREATE INDEX idx_role_permissions_permission_id ON role_permissions(permission_id);

-- =============================================
-- 6. 사용자-시스템역할 매핑
-- =============================================
CREATE TABLE user_system_roles (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id     UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_by UUID REFERENCES users(id),
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, role_id)
);

CREATE INDEX idx_user_system_roles_user_id ON user_system_roles(user_id);

-- =============================================
-- 7. 프로젝트 테이블
-- =============================================
CREATE TABLE projects (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(200) NOT NULL,
    code        VARCHAR(50) NOT NULL UNIQUE,    -- URL-safe 코드
    description TEXT,
    status      VARCHAR(20) NOT NULL DEFAULT 'active'
                CHECK (status IN ('active', 'archived', 'suspended')),
    created_by  UUID REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_projects_code ON projects(code);
CREATE INDEX idx_projects_status ON projects(status);

-- =============================================
-- 8. 프로젝트 멤버 (Temporal: 시작/종료일)
-- =============================================
CREATE TABLE project_members (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    project_id  UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    start_date  DATE NOT NULL DEFAULT CURRENT_DATE,
    end_date    DATE,                           -- NULL이면 무기한
    status      VARCHAR(20) NOT NULL DEFAULT 'active'
                CHECK (status IN ('active', 'inactive', 'expired')),
    assigned_by UUID REFERENCES users(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, project_id)
);

CREATE INDEX idx_project_members_user_id ON project_members(user_id);
CREATE INDEX idx_project_members_project_id ON project_members(project_id);
CREATE INDEX idx_project_members_status ON project_members(status);
CREATE INDEX idx_project_members_end_date ON project_members(end_date)
    WHERE end_date IS NOT NULL;

-- =============================================
-- 9. 프로젝트 멤버-역할 매핑
-- =============================================
CREATE TABLE project_member_roles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_member_id   UUID NOT NULL REFERENCES project_members(id) ON DELETE CASCADE,
    role_id             UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_by         UUID REFERENCES users(id),
    assigned_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(project_member_id, role_id)
);

CREATE INDEX idx_pmr_project_member_id ON project_member_roles(project_member_id);
CREATE INDEX idx_pmr_role_id ON project_member_roles(role_id);

-- =============================================
-- 10. 권한 위임 테이블
-- =============================================
CREATE TABLE permission_delegations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    delegator_id    UUID NOT NULL REFERENCES users(id),     -- 위임자
    delegatee_id    UUID NOT NULL REFERENCES users(id),     -- 수임자
    permission_id   UUID NOT NULL REFERENCES permissions(id),
    project_id      UUID REFERENCES projects(id),           -- 프로젝트 범위 위임 시
    reason          TEXT,
    start_date      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    end_date        TIMESTAMPTZ NOT NULL,                   -- 위임 종료일 (필수)
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'revoked', 'expired')),
    revoked_at      TIMESTAMPTZ,
    revoked_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (end_date > start_date),
    CHECK (delegator_id != delegatee_id)
);

CREATE INDEX idx_delegations_delegator ON permission_delegations(delegator_id);
CREATE INDEX idx_delegations_delegatee ON permission_delegations(delegatee_id);
CREATE INDEX idx_delegations_status ON permission_delegations(status);
CREATE INDEX idx_delegations_end_date ON permission_delegations(end_date)
    WHERE status = 'active';

-- =============================================
-- 11. 승인 요청 테이블
-- =============================================
CREATE TABLE approval_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requester_id    UUID NOT NULL REFERENCES users(id),
    approver_id     UUID REFERENCES users(id),              -- 지정 승인자 (NULL이면 역할 기반)
    request_type    VARCHAR(50) NOT NULL
                    CHECK (request_type IN (
                        'role_assignment',       -- 역할 할당 요청
                        'permission_grant',      -- 권한 부여 요청
                        'project_access',        -- 프로젝트 접근 요청
                        'delegation'             -- 위임 요청
                    )),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'cancelled', 'expired')),
    request_data    JSONB NOT NULL,             -- 요청 상세 (역할ID, 프로젝트ID 등)
    reason          TEXT,                       -- 요청 사유
    decision_comment TEXT,                      -- 승인/거절 사유
    decided_by      UUID REFERENCES users(id),
    decided_at      TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,               -- 요청 만료일
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_approval_requester ON approval_requests(requester_id);
CREATE INDEX idx_approval_approver ON approval_requests(approver_id);
CREATE INDEX idx_approval_status ON approval_requests(status);
CREATE INDEX idx_approval_type_status ON approval_requests(request_type, status);

-- =============================================
-- 12. 감사 로그 테이블 (불변, append-only)
-- =============================================
CREATE TABLE audit_logs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID,                           -- 행위자 (시스템 작업 시 NULL)
    user_email  VARCHAR(255),                   -- 비정규화: 사용자 삭제 후에도 조회 가능
    action      VARCHAR(100) NOT NULL,          -- e.g., "AUTH_LOGIN", "PERM_GRANT"
    category    VARCHAR(20) NOT NULL
                CHECK (category IN ('AUTH', 'PERM', 'ADMIN', 'DATA', 'SYSTEM')),
    target_type VARCHAR(100),                   -- e.g., "user", "role", "project"
    target_id   VARCHAR(255),                   -- 대상 리소스 ID
    result      VARCHAR(20) NOT NULL
                CHECK (result IN ('success', 'failure', 'error')),
    ip_address  INET,
    user_agent  TEXT,
    details     JSONB,                          -- 변경 전/후 데이터 등
    request_id  VARCHAR(100),                   -- 요청 추적용
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 감사 로그는 INSERT만 허용, UPDATE/DELETE 불가
-- 이를 위해 TRIGGER 또는 RLS(Row Level Security) 사용
CREATE INDEX idx_audit_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_action ON audit_logs(action);
CREATE INDEX idx_audit_category ON audit_logs(category);
CREATE INDEX idx_audit_created_at ON audit_logs(created_at);
CREATE INDEX idx_audit_target ON audit_logs(target_type, target_id);
CREATE INDEX idx_audit_result ON audit_logs(result);

-- 감사 로그 불변성 보장 트리거
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'audit_logs 테이블은 수정/삭제할 수 없습니다';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_immutable
    BEFORE UPDATE OR DELETE ON audit_logs
    FOR EACH ROW
    EXECUTE FUNCTION prevent_audit_modification();

-- =============================================
-- 13. Refresh Token 테이블 (서버측 관리)
-- =============================================
CREATE TABLE refresh_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash      VARCHAR(255) NOT NULL UNIQUE,   -- 토큰 해시값 저장
    session_id      UUID NOT NULL,
    device_info     JSONB,                          -- 디바이스/브라우저 정보
    ip_address      INET,
    expires_at      TIMESTAMPTZ NOT NULL,
    is_revoked      BOOLEAN NOT NULL DEFAULT false,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rt_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_rt_token_hash ON refresh_tokens(token_hash);
CREATE INDEX idx_rt_session_id ON refresh_tokens(session_id);
CREATE INDEX idx_rt_expires_at ON refresh_tokens(expires_at)
    WHERE NOT is_revoked;

-- =============================================
-- 14. 세션 테이블 (동시 세션 제어)
-- =============================================
CREATE TABLE sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    device_info     JSONB,                          -- {browser, os, device}
    ip_address      INET,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_activity   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL,           -- 절대 만료 (8시간)
    is_active       BOOLEAN NOT NULL DEFAULT true,
    terminated_by   VARCHAR(50),                    -- 'user', 'admin', 'timeout', 'session_limit'
    terminated_at   TIMESTAMPTZ
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_active ON sessions(user_id, is_active)
    WHERE is_active = true;
CREATE INDEX idx_sessions_expires ON sessions(expires_at)
    WHERE is_active = true;

-- =============================================
-- Temporal 데이터 처리: 만료된 프로젝트 멤버십 자동 비활성화
-- =============================================
-- 스케줄러(Cron Job)로 매일 실행하거나, 권한 체크 시 실시간 확인
CREATE OR REPLACE FUNCTION expire_project_memberships()
RETURNS void AS $$
BEGIN
    UPDATE project_members
    SET status = 'expired', updated_at = NOW()
    WHERE status = 'active'
      AND end_date IS NOT NULL
      AND end_date < CURRENT_DATE;
END;
$$ LANGUAGE plpgsql;

-- 만료된 위임 자동 비활성화
CREATE OR REPLACE FUNCTION expire_delegations()
RETURNS void AS $$
BEGIN
    UPDATE permission_delegations
    SET status = 'expired'
    WHERE status = 'active'
      AND end_date < NOW();
END;
$$ LANGUAGE plpgsql;
```

**Uncle Bob:** DDL이 깔끔합니다. 한 가지 질문: `audit_logs` 테이블에 `user_email`을 비정규화한 이유는?

**Kleppmann:** 감사 로그는 **법적 증거**입니다. 사용자가 나중에 삭제되더라도 "누가 했는지" 알 수 있어야 합니다. `user_id`만 있으면 사용자 삭제 후 JOIN이 불가합니다. 그래서 이메일을 비정규화해서 저장합니다.

**Schneier:** 그리고 `audit_logs` 테이블의 불변성 트리거가 중요합니다. DBA라도 로그를 수정/삭제할 수 없어야 합니다. 운영 환경에서는 이 트리거를 절대 비활성화하면 안 됩니다.

**Fowler:** `project_members`의 temporal 처리를 다시 봅시다. `start_date`와 `end_date`로 기간을 관리하는데, 권한 체크 시 이 날짜를 실시간으로 확인해야 합니다. 쿼리 성능이 걱정되지 않나요?

**Kleppmann:** 두 가지 방법이 있습니다. 첫째, `end_date` 인덱스를 만들어서 쿼리 최적화. 둘째, 스케줄러로 매일 만료된 멤버십을 `expired` 상태로 변경. 권한 체크 시에는 `status = 'active'`만 보면 됩니다. 두 방법을 **병행**하는 게 안전합니다.

> **합의:**
> 1. PostgreSQL 기반 13개 테이블 DDL 확정
> 2. 감사 로그: 비정규화(user_email) + 불변성 트리거
> 3. Temporal 처리: end_date 인덱스 + 스케줄러 병행
> 4. Refresh Token: 해시값 저장 (원본은 클라이언트에만)

---

## 토의 3.4: 권한 체크 API 설계

**Beck (사회자):** API 설계로 넘어갑니다. Uncle Bob, Contract-First로 가시죠.

---

**Uncle Bob:** 권한 관련 API는 5개 그룹입니다. 모든 API는 표준 응답 포맷을 따릅니다.

### 표준 응답 포맷

```typescript
interface ApiResponse<T> {
  status: 'success' | 'error';
  data: T;
  error?: {
    code: string;         // e.g., "AUTH_001", "PERM_003"
    message: string;      // 사용자 표시용 메시지
    details?: any;        // 디버깅용 상세 정보
  };
  metadata?: {
    requestId: string;    // 요청 추적 ID
    timestamp: string;    // ISO 8601
    pagination?: {
      page: number;
      pageSize: number;
      totalCount: number;
      totalPages: number;
    };
  };
}

// 에러 코드 체계
// AUTH_001: 인증 실패 (잘못된 자격증명)
// AUTH_002: 토큰 만료
// AUTH_003: 토큰 무효
// AUTH_004: 세션 초과
// AUTH_005: 계정 잠김
// PERM_001: 권한 없음
// PERM_002: 역할 없음
// PERM_003: 잘못된 권한 형식
// PERM_004: 위임 불가 (본인이 없는 권한)
// PROJ_001: 프로젝트 없음
// PROJ_002: 프로젝트 멤버 아님
// PROJ_003: 프로젝트 비활성
// VAL_001: 입력값 유효성 오류
// SYS_001: 내부 서버 오류
```

### 그룹 1: 인증 API

```typescript
// ===== POST /auth/login =====
interface LoginRequest {
  email: string;
  password: string;
  deviceInfo?: {
    browser: string;
    os: string;
    device: string;
  };
}

interface LoginResponse {
  tokens: AuthTokens;
  user: {
    id: string;
    email: string;
    name: string;
    systemRoles: string[];
    projects: ProjectClaim[];
  };
  sessionId: string;
}
// 에러: AUTH_001 (잘못된 자격증명), AUTH_005 (계정 잠김)

// ===== POST /auth/token/refresh =====
interface TokenRefreshRequest {
  refreshToken: string;     // 또는 HttpOnly Cookie에서 자동 전달
}

interface TokenRefreshResponse {
  tokens: AuthTokens;       // 새 access + refresh token
}
// 에러: AUTH_002 (토큰 만료), AUTH_003 (토큰 무효/폐기됨)

// ===== POST /auth/logout =====
interface LogoutRequest {
  refreshToken?: string;    // 현재 세션의 refresh token
  allSessions?: boolean;    // true면 모든 세션 로그아웃
}
// 응답: 204 No Content (성공)

// ===== GET /auth/me =====
// Authorization: Bearer {accessToken}
interface MeResponse {
  id: string;
  email: string;
  name: string;
  department?: string;
  systemRoles: string[];
  projects: {
    id: string;
    code: string;
    name: string;
    roles: string[];
  }[];
  activeSessions: number;
  lastLoginAt: string;
}
```

### 그룹 2: 권한 체크 API

```typescript
// ===== POST /permissions/check =====
// 단일 권한 체크 (고성능, <10ms 목표)
interface PermissionCheckRequest {
  userId: string;
  permission: string;       // e.g., "master-code:write"
  projectId?: string;       // 프로젝트 범위 체크 시
  context?: {               // ABAC용 추가 컨텍스트
    ip?: string;
    time?: string;
    [key: string]: any;
  };
}

interface PermissionCheckResponse {
  allowed: boolean;
  reason?: string;          // 거부 사유 (디버깅용)
  evaluatedAt: string;      // 판정 시각
  cached: boolean;          // 캐시된 결과인지
}

// ===== POST /permissions/check-batch =====
// 일괄 권한 체크 (UI 렌더링 시 한번에 여러 권한 확인)
interface BatchPermissionCheckRequest {
  userId: string;
  projectId?: string;
  permissions: string[];    // e.g., ["master-code:read", "master-code:write", "user:delete"]
}

interface BatchPermissionCheckResponse {
  results: {
    [permission: string]: {
      allowed: boolean;
      reason?: string;
    };
  };
  evaluatedAt: string;
}

// ===== GET /permissions/my-permissions?projectId={projectId} =====
// 내 권한 목록 조회 (UI "내 권한" 페이지용)
interface MyPermissionsResponse {
  systemPermissions: EffectivePermission[];
  projectPermissions: {
    projectId: string;
    projectName: string;
    permissions: EffectivePermission[];
  }[];
}

interface EffectivePermission {
  permission: string;       // e.g., "master-code:write"
  source: 'role' | 'delegation'; // 권한 출처
  sourceDetail: string;     // e.g., "PROJECT_ADMIN 역할" or "홍길동으로부터 위임"
  expiresAt?: string;       // 위임인 경우 만료일
}
```

**Fowler:** `check-batch` API가 핵심입니다. UI에서 버튼 활성화/비활성화를 위해 10-20개 권한을 한번에 체크해야 합니다. 이걸 개별 API 호출로 하면 N번 호출이 필요한데, batch로 한번에 처리해야 합니다.

**Karpathy:** batch 호출 결과를 **클라이언트에서 캐싱**할 수도 있습니다. 권한이 바뀌는 건 드문 이벤트이니, 5분 정도 캐싱하면 API 호출을 대폭 줄일 수 있습니다.

**Schneier:** 클라이언트 캐싱은 위험합니다. 권한이 박탈된 후에도 5분간 접근 가능하다는 뜻입니다. 서버 측에서 캐싱하되, 권한 변경 이벤트가 발생하면 **캐시 무효화**하는 게 더 안전합니다.

### 그룹 3: 역할 관리 API

```typescript
// ===== CRUD for Roles =====

// POST /roles (역할 생성)
interface CreateRoleRequest {
  name: string;
  scope: 'system' | 'project';
  description: string;
  permissions: string[];      // permission IDs
  parentRoleId?: string;
  projectId?: string;         // project scope인 경우
}

// GET /roles?scope={scope}&projectId={projectId}&page={page}&pageSize={pageSize}
// 역할 목록 조회

// GET /roles/{id}
// 역할 상세 조회 (포함된 권한 목록 포함)

// PUT /roles/{id}
interface UpdateRoleRequest {
  name?: string;
  description?: string;
  permissions?: string[];     // 전체 교체 (PUT semantics)
  parentRoleId?: string | null;
}

// DELETE /roles/{id}
// 역할 삭제 (할당된 사용자가 있으면 거부)

// ===== GET /roles/templates =====
// 시스템 기본 템플릿 목록
interface RoleTemplatesResponse {
  templates: {
    id: string;
    name: string;
    scope: 'system' | 'project';
    description: string;
    permissionCount: number;
    permissions: Permission[];
  }[];
}

// ===== POST /roles/{id}/clone =====
// 역할 복제 (템플릿에서 프로젝트 역할 생성 시)
interface CloneRoleRequest {
  newName: string;
  projectId: string;          // 복제 대상 프로젝트
  addPermissions?: string[];  // 추가할 권한 IDs
  removePermissions?: string[]; // 제거할 권한 IDs
}
```

### 그룹 4: 사용자-역할 할당 API

```typescript
// ===== POST /projects/{projectId}/members =====
// 프로젝트에 멤버 추가
interface AddProjectMemberRequest {
  userId: string;
  roles: string[];            // role IDs
  startDate?: string;         // ISO 8601 date
  endDate?: string;           // temporal: 기간 지정
}

// ===== PUT /projects/{projectId}/members/{userId}/roles =====
// 프로젝트 멤버의 역할 변경
interface UpdateMemberRolesRequest {
  roles: string[];            // 새 역할 IDs (전체 교체)
  endDate?: string;           // 만료일 변경
}

// ===== DELETE /projects/{projectId}/members/{userId} =====
// 프로젝트에서 멤버 제거

// ===== GET /projects/{projectId}/members?page={page}&pageSize={pageSize} =====
// 프로젝트 멤버 목록 조회
interface ProjectMembersResponse {
  members: {
    userId: string;
    userName: string;
    email: string;
    roles: { id: string; name: string }[];
    startDate: string;
    endDate?: string;
    status: string;
  }[];
}
```

### 그룹 5: 위임 API

```typescript
// ===== POST /delegations =====
// 권한 위임 생성
interface CreateDelegationRequest {
  delegateeId: string;        // 수임자 ID
  permissionIds: string[];    // 위임할 권한 IDs
  projectId?: string;         // 프로젝트 범위 위임 시
  reason: string;             // 위임 사유 (필수)
  startDate?: string;         // 기본: 즉시
  endDate: string;            // 위임 종료일 (필수)
}

// 제약사항: 본인이 가진 권한만 위임 가능

// ===== PUT /delegations/{id}/revoke =====
// 위임 철회
interface RevokeDelegationRequest {
  reason?: string;
}

// ===== GET /delegations?type={given|received}&status={active|all} =====
// 위임 목록 조회 (내가 준 것 / 받은 것)
```

### 그룹 6: 승인 워크플로우 API

```typescript
// ===== POST /approval-requests =====
// 승인 요청 생성
interface CreateApprovalRequest {
  requestType: 'role_assignment' | 'permission_grant' | 'project_access' | 'delegation';
  requestData: {
    roleId?: string;
    permissionIds?: string[];
    projectId?: string;
    delegateeId?: string;
    startDate?: string;
    endDate?: string;
  };
  reason: string;             // 요청 사유
  approverId?: string;        // 지정 승인자 (없으면 프로젝트 관리자)
}

// ===== PUT /approval-requests/{id}/approve =====
interface ApproveRequest {
  comment?: string;           // 승인 코멘트
}

// ===== PUT /approval-requests/{id}/reject =====
interface RejectRequest {
  comment: string;            // 거절 사유 (필수)
}

// ===== GET /approval-requests?status={pending|all}&role={requester|approver} =====
// 승인 요청 목록 (내가 요청한 것 / 내가 승인해야 하는 것)
interface ApprovalListResponse {
  requests: {
    id: string;
    requestType: string;
    requester: { id: string; name: string; email: string };
    approver?: { id: string; name: string; email: string };
    status: string;
    reason: string;
    requestData: any;
    createdAt: string;
    decidedAt?: string;
    decisionComment?: string;
  }[];
}
```

**Fowler:** API 설계가 RESTful 원칙을 잘 따르고 있습니다. 한 가지 추가할 점: 모든 상태 변경 API(POST, PUT, DELETE)는 자동으로 **감사 로그**를 남겨야 합니다. API Gateway 레벨에서 처리하는 게 좋을까요, 각 서비스에서 처리하는 게 좋을까요?

**Uncle Bob:** 감사 로그는 **도메인 레벨**에서 남겨야 합니다. API Gateway에서 남기면 비즈니스 컨텍스트(무엇이 바뀌었는지)를 담을 수 없습니다. 서비스 내부에서 이벤트를 발행하고, Audit Service가 그걸 구독해서 저장하는 패턴이 좋습니다.

> **합의:**
> 1. 5개 API 그룹 (인증, 권한체크, 역할관리, 사용자-역할, 위임+승인)
> 2. 표준 에러 코드 체계 (AUTH_xxx, PERM_xxx, PROJ_xxx, VAL_xxx, SYS_xxx)
> 3. batch 권한 체크 API로 UI 성능 최적화
> 4. 감사 로그는 도메인 레벨에서 이벤트 발행 -> Audit Service 구독 패턴

---

## 토의 3.5: Policy Engine 연동 설계 (OPA)

**Beck (사회자):** OPA(Open Policy Agent) 연동을 구체화합시다. Fowler, 아키텍처부터 설명해주시죠.

---

**Fowler:** OPA 연동 아키텍처는 크게 세 가지 패턴이 있습니다:

```
패턴 1: API Gateway + OPA Sidecar
  클라이언트 -> API Gateway -> OPA(사이드카) -> 백엔드 서비스
  장점: 중앙 집중 정책 평가
  단점: Gateway가 SPOF, 모든 요청에 지연 추가

패턴 2: 각 서비스 + OPA Library
  클라이언트 -> 서비스(내장 OPA) -> DB
  장점: 서비스 자율성, 네트워크 지연 없음
  단점: 각 서비스가 OPA를 내장해야 함

패턴 3: Hybrid (권장)
  - API Gateway: 기본 인증/인가 (토큰 유효성, 기본 역할 체크)
  - 각 서비스: 세밀한 비즈니스 권한 (OPA 사이드카 또는 라이브러리)
```

저는 **패턴 3 Hybrid**를 추천합니다.

```
요청 흐름:

[클라이언트]
    |
    v
[API Gateway]
    |-- 1단계: JWT 토큰 검증 (서명, 만료)
    |-- 2단계: 기본 접근 제어 (URL 패턴 + 역할 매칭)
    |   예: /admin/* -> SYSTEM_ADMIN 역할 필요
    |
    v
[백엔드 서비스]
    |-- 3단계: OPA 사이드카에 세부 권한 질의
    |   예: "이 사용자가 project-a의 master-code를 approve할 수 있는가?"
    |
    v
[OPA Sidecar]
    |-- Policy Bundle에서 정책 로드
    |-- Input(사용자, 리소스, 액션, 컨텍스트) 평가
    |-- 결과 반환: allow/deny + 사유
```

**Uncle Bob:** OPA의 정책 언어인 **Rego**로 실제 정책을 작성해봅시다. 핵심 시나리오 몇 가지를 Rego로 표현합니다.

### Policy Bundle 구조

```
policies/
├── auth-policies/
│   ├── token_validation.rego    # 토큰 유효성 정책
│   └── session_policy.rego      # 세션 관련 정책
├── project-policies/
│   ├── project_access.rego      # 프로젝트 접근 정책
│   └── member_permission.rego   # 프로젝트 멤버 권한 정책
├── master-data-policies/
│   ├── code_management.rego     # 기준정보 코드 관리 정책
│   └── approval_policy.rego     # 승인 워크플로우 정책
└── data.json                    # 정적 데이터 (역할-권한 매핑 등)
```

### Rego 정책 예시

```rego
# ===== project_access.rego =====
package project.access

import future.keywords.if
import future.keywords.in

default allow := false

# System Admin은 모든 프로젝트 접근 가능
allow if {
    "SYSTEM_ADMIN" in input.user.systemRoles
}

# Super Admin은 모든 권한
allow if {
    "SUPER_ADMIN" in input.user.systemRoles
}

# 프로젝트 멤버는 해당 프로젝트 접근 가능
allow if {
    some project in input.user.projects
    project.id == input.resource.projectId
    project_membership_active(project)
}

# 멤버십 활성 상태 확인
project_membership_active(project) if {
    project.status == "active"
}

# ===== member_permission.rego =====
package project.permission

import future.keywords.if
import future.keywords.in

default allow := false

# 사용자의 프로젝트 역할에 해당 권한이 있는지 확인
allow if {
    some project in input.user.projects
    project.id == input.resource.projectId

    some role in project.roles
    some permission in data.role_permissions[role]

    permission.resource == input.resource.type
    permission.action == input.action
}

# 위임받은 권한 확인
allow if {
    some delegation in input.user.delegations
    delegation.projectId == input.resource.projectId
    delegation.permission.resource == input.resource.type
    delegation.permission.action == input.action
    delegation.status == "active"

    # 위임 기간 확인
    time.now_ns() >= time.parse_rfc3339_ns(delegation.startDate)
    time.now_ns() <= time.parse_rfc3339_ns(delegation.endDate)
}

# 거부 사유 생성
reason := msg if {
    not allow
    msg := sprintf("사용자 %s에게 %s:%s@%s 권한이 없습니다", [
        input.user.id,
        input.resource.type,
        input.action,
        input.resource.projectId
    ])
}

# ===== approval_policy.rego =====
package master.approval

import future.keywords.if
import future.keywords.in

default requires_approval := false

# 기준정보 코드 수정은 승인 필요
requires_approval if {
    input.resource.type == "master-code"
    input.action == "write"
}

# 기준정보 코드 삭제는 승인 필요
requires_approval if {
    input.resource.type == "master-code"
    input.action == "delete"
}

# 승인 가능한 사용자인지 확인
can_approve if {
    some project in input.approver.projects
    project.id == input.resource.projectId

    some role in project.roles
    role == "PROJECT_ADMIN"
}

# 자기 자신의 요청은 승인 불가
can_approve := false if {
    input.approver.id == input.requester.id
}
```

### OPA 연동 TypeScript 클라이언트

```typescript
// ===== OPA Client =====
interface OPAInput {
  user: {
    id: string;
    systemRoles: string[];
    projects: ProjectClaim[];
    delegations?: DelegationInfo[];
  };
  resource: {
    type: string;       // e.g., "master-code"
    projectId?: string;
    id?: string;        // 특정 리소스 인스턴스
  };
  action: string;       // e.g., "write"
  context?: {
    ip?: string;
    time?: string;
    department?: string;
  };
}

interface OPAResult {
  allow: boolean;
  reason?: string;
  requiresApproval?: boolean;
}

class OPAClient {
  private baseUrl: string;      // OPA sidecar URL (http://localhost:8181)
  private cache: Map<string, { result: OPAResult; expiresAt: number }>;
  private cacheTTL: number;     // 기본 60초

  async checkPermission(input: OPAInput): Promise<OPAResult> {
    const cacheKey = this.buildCacheKey(input);

    // 캐시 확인
    const cached = this.cache.get(cacheKey);
    if (cached && cached.expiresAt > Date.now()) {
      return { ...cached.result, cached: true };
    }

    try {
      // OPA에 질의
      const response = await fetch(`${this.baseUrl}/v1/data/project/permission`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ input }),
      });

      const data = await response.json();
      const result: OPAResult = {
        allow: data.result?.allow ?? false,
        reason: data.result?.reason,
        requiresApproval: data.result?.requires_approval,
      };

      // 캐시 저장
      this.cache.set(cacheKey, {
        result,
        expiresAt: Date.now() + this.cacheTTL,
      });

      return result;
    } catch (error) {
      // OPA 장애 시 Fallback
      return this.fallbackCheck(input);
    }
  }

  // OPA 장애 시 Fallback: DB에서 직접 권한 체크
  private async fallbackCheck(input: OPAInput): Promise<OPAResult> {
    console.warn('OPA unavailable, falling back to DB check');
    // DB 기반 단순 RBAC 체크 (ABAC 조건 무시)
    // ... fallback 로직
    return { allow: false, reason: 'OPA unavailable, denied by default' };
  }

  // 권한 변경 시 캐시 무효화
  invalidateCache(userId?: string, projectId?: string): void {
    if (!userId && !projectId) {
      this.cache.clear();
      return;
    }
    // 특정 사용자/프로젝트 관련 캐시만 삭제
    for (const [key] of this.cache) {
      if (userId && key.includes(userId)) this.cache.delete(key);
      if (projectId && key.includes(projectId)) this.cache.delete(key);
    }
  }
}
```

**Schneier:** Fallback 정책이 중요합니다. OPA가 죽었을 때 **기본 거부(deny by default)**인지, **기본 허용**인지. 보안 관점에서는 반드시 **기본 거부**여야 합니다.

**Fowler:** 동의합니다. 다만 읽기 전용(read) 작업에 대해서는 **DB Fallback**으로 기본 RBAC 체크를 수행하는 게 가용성 측면에서 합리적입니다. 쓰기(write/delete/approve) 작업은 OPA 장애 시 거부합니다.

```
OPA 장애 시 Fallback 전략:
  read 작업  -> DB 기반 단순 RBAC 체크 (ABAC 조건 무시)
  write 작업 -> 기본 거부 (deny by default)
  approve/delete -> 기본 거부 (deny by default)
```

**Ng:** OPA 결정 캐시 TTL은 어떻게 설정하나요?

**Fowler:** 세 가지 레벨입니다:

```
캐시 TTL 설정:
  - Permission Check 결과: 60초 (권한 변경 이벤트로 즉시 무효화 가능)
  - Policy Bundle 동기화: 30초 (OPA가 Policy Server에서 번들 갱신)
  - Role-Permission 매핑 데이터: 5분 (data.json 갱신)
```

> **합의:**
> 1. Hybrid 패턴: API Gateway(기본 인증) + 서비스별 OPA Sidecar(세부 인가)
> 2. Policy Bundle 3분류: auth-policies, project-policies, master-data-policies
> 3. OPA 장애 Fallback: read는 DB RBAC, write/delete/approve는 기본 거부
> 4. 캐시 TTL: 결정 60초, 번들 30초, 데이터 5분 + 이벤트 기반 무효화

---

## 토의 3.6: 권한 관리 UI/UX 설계

**Beck (사회자):** UI/UX 설계로 넘어갑시다. Nielsen, 리드해주세요.

---

**Nielsen:** 권한 관리 UI는 **세 가지 사용자 유형**을 고려해야 합니다:

```
1. 시스템 관리자: 전체 역할/권한 관리
2. 프로젝트 관리자: 프로젝트 내 멤버/역할 관리
3. 일반 사용자: 본인 권한 확인, 권한 요청, 위임
```

각 유형별 핵심 화면을 설계합니다.

### 화면 1: 역할 관리 (시스템/프로젝트 관리자)

```
+------------------------------------------------------------------+
| 역할 관리                                          [+ 새 역할]    |
+------------------------------------------------------------------+
| 범위: [시스템 v] [프로젝트 v]      검색: [_______________] [검색] |
+------------------------------------------------------------------+
| 역할명          | 범위    | 유형   | 권한 수 | 할당 사용자 | 액션 |
+------------------------------------------------------------------+
| SUPER_ADMIN     | 시스템  | 기본   | 45      | 2           | [조회]|
| SYSTEM_ADMIN    | 시스템  | 기본   | 32      | 5           | [조회]|
| SYSTEM_AUDITOR  | 시스템  | 기본   | 8       | 3           | [조회]|
| PROJECT_ADMIN   | 프로젝트| 템플릿 | 28      | -           | [복제]|
| PROJECT_MEMBER  | 프로젝트| 템플릿 | 15      | -           | [복제]|
| 프로젝트A-개발자 | 프로젝트| 커스텀 | 18      | 12          | [편집]|
+------------------------------------------------------------------+
```

### 화면 2: Permission Matrix (역할 x 리소스 그리드)

```
+------------------------------------------------------------------+
| 역할 권한 매트릭스: [프로젝트A-개발자]              [저장] [초기화] |
+------------------------------------------------------------------+
| 리소스 \ 액션    | Read | Write | Delete | Approve | Export      |
+------------------------------------------------------------------+
| master-code     | [v]  | [v]   | [ ]    | [ ]     | [v]         |
| user            | [v]  | [ ]   | [ ]    | [ ]     | [ ]         |
| audit-log       | [v]  | [ ]   | [ ]    | [ ]     | [v]         |
| dashboard       | [v]  | [ ]   | [ ]    | [ ]     | [ ]         |
| report          | [v]  | [ ]   | [ ]    | [ ]     | [v]         |
| project-config  | [v]  | [ ]   | [ ]    | [ ]     | [ ]         |
| role            | [v]  | [ ]   | [ ]    | [ ]     | [ ]         |
+------------------------------------------------------------------+
| 상속된 권한(PROJECT_MEMBER)은 회색 체크로 표시                     |
| [v] = 직접 할당  [v] = 상속됨 (읽기 전용)                         |
+------------------------------------------------------------------+
```

**Uncle Bob:** Permission Matrix가 핵심 UI입니다. 관리자가 한눈에 역할의 권한 범위를 파악하고 수정할 수 있습니다. 상속된 권한은 **회색 체크박스(비활성)**으로 표시해서 직접 할당과 구분하는 게 중요합니다.

### 화면 3: 내 권한 조회 (일반 사용자)

```
+------------------------------------------------------------------+
| 내 권한                                                           |
+------------------------------------------------------------------+
| 프로젝트: [전체 v]                                                |
+------------------------------------------------------------------+
|                                                                    |
| [시스템 권한]                                                      |
|   역할: SYSTEM_AUDITOR                                            |
|   - audit-log:read    (전체 감사 로그 조회)                        |
|   - user:read         (사용자 정보 조회)                           |
|                                                                    |
| [프로젝트 A]  역할: 프로젝트A-개발자                               |
|   - master-code:read  (기준정보 코드 조회)          [직접]         |
|   - master-code:write (기준정보 코드 수정)          [직접]         |
|   - master-code:export(기준정보 코드 내보내기)      [직접]         |
|   - master-code:approve(기준정보 코드 승인)         [위임]         |
|     └ 홍길동 → 나, 2025-02-01 ~ 2025-02-28       [위임 상세]     |
|                                                                    |
| [프로젝트 B]  역할: PROJECT_VIEWER                                |
|   - dashboard:read    (대시보드 조회)               [직접]         |
|   - report:read       (보고서 조회)                 [직접]         |
|                                                                    |
| [접근 가능한 프로젝트가 없습니다 → [프로젝트 접근 요청] ]          |
+------------------------------------------------------------------+
```

### 화면 4: 역할 비교 (Side-by-side Diff)

```
+------------------------------------------------------------------+
| 역할 비교                                                         |
+------------------------------------------------------------------+
| 비교 대상: [프로젝트A-개발자 v]  vs  [프로젝트A-관리자 v]          |
+------------------------------------------------------------------+
| 리소스:액션          | 프로젝트A-개발자  | 프로젝트A-관리자         |
+------------------------------------------------------------------+
| master-code:read    |       v           |       v                  |
| master-code:write   |       v           |       v                  |
| master-code:delete  |       -           |       v          [차이]  |
| master-code:approve |       -           |       v          [차이]  |
| user:read           |       v           |       v                  |
| user:write          |       -           |       v          [차이]  |
| user:delete         |       -           |       v          [차이]  |
| role:read           |       v           |       v                  |
| role:write          |       -           |       v          [차이]  |
| project-config:write|       -           |       v          [차이]  |
+------------------------------------------------------------------+
| 요약: 프로젝트A-관리자가 6개 권한을 추가로 보유                    |
+------------------------------------------------------------------+
```

**Nielsen:** 역할 비교는 **역할 변경을 요청할 때** 특히 유용합니다. "현재 역할과 요청 역할의 차이가 이것입니다"를 시각적으로 보여줄 수 있습니다.

### 화면 5: 권한 요청 워크플로우 UI

```
+------------------------------------------------------------------+
| 권한 요청                                                         |
+------------------------------------------------------------------+
| 요청 유형: ( ) 프로젝트 접근 요청                                  |
|           (o) 역할 변경 요청                                       |
|           ( ) 추가 권한 요청                                       |
|           ( ) 권한 위임 요청                                       |
+------------------------------------------------------------------+
| 프로젝트: [프로젝트 A  v]                                         |
| 현재 역할: PROJECT_MEMBER                                         |
| 요청 역할: [프로젝트A-개발자 v]  [역할 비교 보기]                  |
+------------------------------------------------------------------+
| 요청 사유:                                                        |
| [기준정보 코드 수정 업무가 추가되어 개발자 역할이 필요합니다.    ] |
|                                                                    |
| 승인자: 김관리 (프로젝트 A 관리자)   [자동 지정]                   |
|                                                                    |
|                                [취소]  [요청 제출]                 |
+------------------------------------------------------------------+
```

### 화면 6: 위임 UI

```
+------------------------------------------------------------------+
| 권한 위임                                                         |
+------------------------------------------------------------------+
| 수임자: [__________] [사용자 검색]                                |
|   선택됨: 이영희 (이영희@example.com)                              |
+------------------------------------------------------------------+
| 위임할 권한:                                                      |
| 프로젝트: [프로젝트 A v]                                          |
|                                                                    |
| [ ] master-code:read    (기준정보 코드 조회)                       |
| [v] master-code:write   (기준정보 코드 수정)                       |
| [v] master-code:approve (기준정보 코드 승인)                       |
| [ ] dashboard:read      (대시보드 조회)                            |
+------------------------------------------------------------------+
| 위임 기간:                                                        |
| 시작: [2025-02-01]  종료: [2025-02-28]                            |
+------------------------------------------------------------------+
| 사유: [출장 기간 중 코드 승인 업무 위임                         ]  |
|                                                                    |
|                                [취소]  [위임 요청]                 |
+------------------------------------------------------------------+
```

### 화면 7: 대량 작업 (CSV Import)

```
+------------------------------------------------------------------+
| 사용자-역할 일괄 등록                                              |
+------------------------------------------------------------------+
| [CSV 템플릿 다운로드]                                              |
|                                                                    |
| CSV 형식:                                                          |
| email,project_code,role_name,start_date,end_date                  |
| user1@example.com,proj-a,PROJECT_MEMBER,2025-01-01,2025-12-31    |
| user2@example.com,proj-a,PROJECT_ADMIN,2025-01-01,               |
|                                                                    |
| [파일 선택: import_members.csv]  [미리보기]                        |
+------------------------------------------------------------------+
| 미리보기 결과:                                                     |
| 총 50건 | 성공 예상: 47건 | 오류: 3건                              |
|                                                                    |
| 오류 항목:                                                         |
| 행 12: user5@example.com - 존재하지 않는 사용자                    |
| 행 23: proj-x - 존재하지 않는 프로젝트                             |
| 행 41: INVALID_ROLE - 존재하지 않는 역할                           |
|                                                                    |
|                    [오류 항목 제외하고 실행]  [전체 취소]           |
+------------------------------------------------------------------+
```

### 화면 8: 대시보드 (권한 통계)

```
+------------------------------------------------------------------+
| 권한 관리 대시보드                                                 |
+------------------------------------------------------------------+
|  [활성 사용자]    [역할 수]     [대기 중 요청]   [오늘 변경사항]    |
|    1,234          28            7                15               |
+------------------------------------------------------------------+
|                                                                    |
| [최근 권한 변경] (최근 7일)                                        |
| +--------------------------------------------------------------+  |
| | 날짜       | 대상       | 변경 내용          | 수행자         |  |
| | 2025-01-29 | 이영희     | 역할 추가:개발자   | 김관리         |  |
| | 2025-01-29 | 박철수     | 프로젝트 B 멤버 추가| 시스템        |  |
| | 2025-01-28 | 최지원     | 역할 제거:ADMIN    | 김관리         |  |
| +--------------------------------------------------------------+  |
|                                                                    |
| [대기 중 승인 요청]                                                |
| +--------------------------------------------------------------+  |
| | 요청자     | 유형           | 요청일     | [승인] [거절]     |  |
| | 홍길동     | 역할 변경 요청  | 2025-01-28 | [승인] [거절]     |  |
| | 김민수     | 프로젝트 접근   | 2025-01-27 | [승인] [거절]     |  |
| +--------------------------------------------------------------+  |
|                                                                    |
| [이상 활동 감지]                                                   |
| - 김의심: 30분 내 5회 권한 거부 발생 (비정상 접근 시도 의심)       |
| - 이탈퇴: 비활성 상태인데 로그인 시도 감지                         |
+------------------------------------------------------------------+
```

**Schneier:** 대시보드의 **이상 활동 감지**가 중요합니다. 단순 통계를 넘어서, 보안 위협 패턴을 감지해야 합니다:

```
이상 활동 탐지 규칙:
1. 짧은 시간 내 다수 권한 거부 (brute-force 시도)
2. 비정상 시간대 로그인 (새벽 2-5시)
3. 비정상 IP에서 접근
4. 비활성 계정 로그인 시도
5. 동시에 다른 지역에서 로그인 (불가능한 이동)
6. 대량 데이터 조회/내보내기 (데이터 유출 시도)
```

**Karpathy:** Phase 2에서 이 규칙들을 ML 모델로 확장할 수 있습니다. 사용자별 행동 패턴을 학습해서 이상 점수(anomaly score)를 계산하는 겁니다. 하지만 Phase 1에서는 규칙 기반으로 충분합니다.

> **합의:**
> 1. 8개 핵심 화면: 역할관리, Permission Matrix, 내 권한, 역할비교, 요청 워크플로우, 위임, CSV Import, 대시보드
> 2. Permission Matrix는 상속 권한을 시각적으로 구분
> 3. 역할 비교(diff)로 변경 요청 시 차이 명확화
> 4. CSV Import에 사전 검증(미리보기) 포함
> 5. 대시보드에 이상 활동 감지 규칙 포함 (Phase 1: 규칙 기반, Phase 2: ML 확장)

---

## 토의 3.7: 감사 로그 설계 (Audit Trail)

**Beck (사회자):** 마지막 토의입니다. 감사 로그는 보안과 컴플라이언스의 핵심입니다. Schneier와 Kleppmann, 함께 리드해주시죠.

---

**Schneier:** 감사 로그의 원칙은 세 가지입니다:

```
1. 불변성 (Immutability): 한번 쓰면 수정/삭제 불가
2. 완전성 (Completeness): 모든 보안 관련 이벤트를 빠짐없이 기록
3. 검색성 (Searchability): 사고 발생 시 빠르게 조회 가능
```

### 로그 스키마 상세

```typescript
interface AuditLogEntry {
  id: string;                  // UUID
  timestamp: string;           // ISO 8601 with timezone

  // WHO - 누가
  actor: {
    userId: string | null;     // 시스템 작업 시 null
    email: string;
    name: string;
    sessionId?: string;
    roles: string[];           // 행위 시점의 역할
  };

  // WHAT - 무엇을
  action: string;              // 액션 코드 (아래 카테고리 참조)
  category: AuditCategory;

  // WHERE - 어디서
  source: {
    ip: string;
    userAgent: string;
    service: string;           // 요청을 처리한 서비스명
    endpoint: string;          // API endpoint
  };

  // TARGET - 대상
  target: {
    type: string;              // "user", "role", "project", "permission"
    id: string;
    name?: string;             // 비정규화
    projectId?: string;
  };

  // RESULT - 결과
  result: 'success' | 'failure' | 'error';
  resultDetail?: string;       // 실패/에러 사유

  // DETAILS - 상세
  details: {
    before?: any;              // 변경 전 상태
    after?: any;               // 변경 후 상태
    metadata?: any;            // 추가 정보
  };

  // TRACKING
  requestId: string;           // 분산 추적용
  correlationId?: string;      // 관련 로그 그룹핑
}

type AuditCategory = 'AUTH' | 'PERM' | 'ADMIN' | 'DATA' | 'SYSTEM';
```

### 액션 코드 카테고리

```typescript
// AUTH 카테고리: 인증 관련
const AUTH_ACTIONS = {
  AUTH_LOGIN_SUCCESS: '로그인 성공',
  AUTH_LOGIN_FAILURE: '로그인 실패',
  AUTH_LOGOUT: '로그아웃',
  AUTH_TOKEN_REFRESH: '토큰 갱신',
  AUTH_TOKEN_REVOKE: '토큰 폐기',
  AUTH_SESSION_TIMEOUT: '세션 타임아웃',
  AUTH_SESSION_TERMINATED: '세션 강제 종료',
  AUTH_PASSWORD_CHANGE: '비밀번호 변경',
  AUTH_ACCOUNT_LOCKED: '계정 잠금',
  AUTH_ACCOUNT_UNLOCKED: '계정 잠금 해제',
} as const;

// PERM 카테고리: 권한 변경 관련
const PERM_ACTIONS = {
  PERM_ROLE_CREATED: '역할 생성',
  PERM_ROLE_UPDATED: '역할 수정',
  PERM_ROLE_DELETED: '역할 삭제',
  PERM_ROLE_ASSIGNED: '역할 할당',
  PERM_ROLE_REVOKED: '역할 박탈',
  PERM_DELEGATION_CREATED: '권한 위임',
  PERM_DELEGATION_REVOKED: '위임 철회',
  PERM_DELEGATION_EXPIRED: '위임 만료',
  PERM_CHECK_DENIED: '권한 체크 거부',
  PERM_REQUEST_CREATED: '권한 요청',
  PERM_REQUEST_APPROVED: '권한 요청 승인',
  PERM_REQUEST_REJECTED: '권한 요청 거절',
} as const;

// ADMIN 카테고리: 관리 작업 관련
const ADMIN_ACTIONS = {
  ADMIN_USER_CREATED: '사용자 생성',
  ADMIN_USER_UPDATED: '사용자 수정',
  ADMIN_USER_DEACTIVATED: '사용자 비활성화',
  ADMIN_PROJECT_CREATED: '프로젝트 생성',
  ADMIN_PROJECT_UPDATED: '프로젝트 수정',
  ADMIN_PROJECT_ARCHIVED: '프로젝트 보관',
  ADMIN_CONFIG_CHANGED: '시스템 설정 변경',
  ADMIN_MEMBER_ADDED: '프로젝트 멤버 추가',
  ADMIN_MEMBER_REMOVED: '프로젝트 멤버 제거',
} as const;
```

**Kleppmann:** 저장소 설계를 논의합시다. 감사 로그는 **메인 DB와 분리**하는 게 좋습니다. 이유는 세 가지입니다:

```
1. 성능: 감사 로그 write가 메인 트랜잭션에 영향 주지 않음
2. 보존: 메인 DB와 다른 보존 정책 적용 가능 (3년+)
3. 보안: 감사 로그 DB에 대한 접근 권한을 별도로 관리
```

아키텍처:

```
[서비스들] --이벤트 발행--> [Message Queue (Kafka/RabbitMQ)]
                                    |
                                    v
                           [Audit Service]
                                    |
                    +---------------+---------------+
                    |               |               |
                    v               v               v
             [Audit DB]     [Search Index]    [Cold Storage]
             (PostgreSQL)   (Elasticsearch)   (S3/MinIO)
             최근 6개월      최근 1년 검색용    3년+ 보관
```

**Schneier:** 이 아키텍처가 맞습니다. 비동기 이벤트 기반이니 서비스 성능에 영향이 없고, 여러 저장소에 분산 저장해서 데이터 손실 위험도 줄입니다.

### Retention 정책

```typescript
interface RetentionPolicy {
  hotStorage: {
    backend: 'postgresql';
    duration: '6months';       // 최근 6개월: 상세 데이터, 빠른 쿼리
    indexing: 'full';
  };
  warmStorage: {
    backend: 'elasticsearch';
    duration: '1year';         // 최근 1년: 검색 최적화
    indexing: 'full-text';
  };
  coldStorage: {
    backend: 's3';
    duration: '3years+';       // 3년 이상: 컴플라이언스 보관
    format: 'parquet';         // 압축 저장
    indexing: 'metadata-only';
  };
  regulations: {
    default: '3years';
    financial: '5years';       // 금융 관련
    gdpr: 'user-request';     // GDPR 삭제 요청 시 익명화
  };
}
```

### 감사 로그 조회 API

```typescript
// ===== GET /audit-logs =====
interface AuditLogQueryParams {
  // 기간 필터
  startDate: string;           // ISO 8601 (필수)
  endDate: string;             // ISO 8601 (필수)

  // 필터
  userId?: string;
  category?: AuditCategory;
  action?: string;
  targetType?: string;
  targetId?: string;
  projectId?: string;
  result?: 'success' | 'failure' | 'error';
  ip?: string;

  // 페이징
  page?: number;               // 기본 1
  pageSize?: number;           // 기본 50, 최대 200

  // 정렬
  sortBy?: 'timestamp' | 'action' | 'user';
  sortOrder?: 'asc' | 'desc';
}

interface AuditLogQueryResponse {
  logs: AuditLogEntry[];
  pagination: {
    page: number;
    pageSize: number;
    totalCount: number;
    totalPages: number;
  };
  summary: {
    totalEvents: number;
    successCount: number;
    failureCount: number;
    topActions: { action: string; count: number }[];
    topUsers: { userId: string; name: string; count: number }[];
  };
}

// ===== GET /audit-logs/export =====
interface AuditLogExportRequest {
  format: 'csv' | 'pdf' | 'json';
  filters: AuditLogQueryParams;
  includeDetails: boolean;     // 상세 JSON 포함 여부
}

// Export는 비동기 처리 (대용량 데이터)
interface AuditLogExportResponse {
  exportId: string;
  status: 'processing' | 'ready' | 'failed';
  downloadUrl?: string;        // 준비 완료 시 다운로드 URL
  expiresAt?: string;          // 다운로드 링크 만료 시간
}

// ===== GET /audit-logs/statistics =====
interface AuditStatisticsRequest {
  startDate: string;
  endDate: string;
  projectId?: string;
  granularity: 'hour' | 'day' | 'week' | 'month';
}

interface AuditStatisticsResponse {
  timeline: {
    period: string;
    totalEvents: number;
    authEvents: number;
    permEvents: number;
    adminEvents: number;
    failureEvents: number;
  }[];
  anomalies: {
    type: string;              // "excessive_denials", "unusual_time", etc.
    description: string;
    severity: 'low' | 'medium' | 'high' | 'critical';
    userId?: string;
    detectedAt: string;
    details: any;
  }[];
}
```

### SIEM 통합

```typescript
// SIEM(Security Information and Event Management) 연동
interface SIEMIntegration {
  // Syslog 형식으로 실시간 전송
  syslog: {
    enabled: boolean;
    host: string;
    port: number;
    protocol: 'tcp' | 'udp' | 'tls';
    format: 'CEF' | 'LEEF' | 'RFC5424';
  };

  // Webhook 기반 알림
  webhook: {
    enabled: boolean;
    url: string;
    headers: Record<string, string>;
    events: string[];          // 전송할 이벤트 필터
    batchSize: number;         // 배치 전송 크기
    flushInterval: number;     // 배치 전송 주기 (초)
  };

  // Kafka 토픽 전송
  kafka: {
    enabled: boolean;
    brokers: string[];
    topic: string;
    compression: 'gzip' | 'snappy' | 'lz4';
  };
}
```

**Ng:** SIEM 통합에서 **CEF(Common Event Format)** 형식은 Splunk, ArcSight 등 주요 SIEM 제품이 지원하니 좋은 선택입니다. Webhook은 Slack/Teams 알림용으로도 활용할 수 있습니다.

**Fowler:** 감사 로그 서비스의 가용성이 중요합니다. 이벤트 큐(Kafka)를 중간에 두면, Audit Service가 일시적으로 죽어도 이벤트가 유실되지 않습니다. Kafka의 retention을 7일로 설정하면 충분한 버퍼입니다.

**Schneier:** 마지막으로, 감사 로그 자체의 **무결성 검증** 메커니즘도 필요합니다. 각 로그 엔트리에 이전 엔트리의 해시를 포함하는 **해시 체인** 방식을 적용하면, 중간에 로그가 조작되었는지 탐지할 수 있습니다.

```typescript
interface AuditLogWithIntegrity extends AuditLogEntry {
  sequenceNumber: number;      // 순서 번호
  previousHash: string;        // 이전 로그의 해시
  hash: string;                // 현재 로그의 해시 (이전해시 + 현재데이터)
}
// 주기적으로 해시 체인 무결성 검증 배치 실행
```

**Kleppmann:** 해시 체인은 좋은 아이디어인데, 성능 비용이 있습니다. 모든 로그에 적용하면 write 시 직전 로그를 읽어야 합니다. **시간 단위 블록 해시**가 더 실용적입니다. 1시간 치 로그를 묶어서 블록 해시를 생성하는 겁니다.

**Schneier:** 동의합니다. 시간 블록 해시가 현실적입니다.

> **합의:**
> 1. 감사 로그는 메인 DB와 분리, 비동기 이벤트 기반 수집
> 2. 3-tier 저장: Hot(PostgreSQL, 6개월) + Warm(Elasticsearch, 1년) + Cold(S3, 3년+)
> 3. 불변성: DB 트리거 + 시간 블록 해시 무결성 검증
> 4. SIEM 통합: Syslog(CEF) + Webhook + Kafka 지원
> 5. Export: CSV/PDF/JSON 비동기 생성 + 다운로드 링크
> 6. 이상 활동 탐지: 규칙 기반 anomaly detection (Phase 1)

---

## 핵심 결정 사항 요약

| # | 결정 사항 | 내용 |
|---|----------|------|
| 1 | 권한 모델 | System Role(전역) + Project Role(프로젝트 범위) 2축 구조. Permission 형식: `resource:action`, Project Scope: `resource:action@project` |
| 2 | Role Template | 시스템 기본 템플릿 제공, 프로젝트별 clone + 커스터마이징. Role Hierarchy(parentRoleId) 지원 |
| 3 | ABAC 확장 | PolicyCondition 인터페이스 Phase 1 정의, Phase 2 OPA 연동 시 활성화 |
| 4 | 인증 흐름 | OAuth2 Authorization Code + PKCE (Phase 1). BFF 패턴은 Phase 2 검토 |
| 5 | Token 관리 | Access Token(JWT, 15분, 메모리), Refresh Token(Opaque, 7일, HttpOnly Cookie, 서버저장). Rotation + 탈취 감지 |
| 6 | Session 관리 | 절대 8시간, 유휴 30분, 동시 3세션, 초과 시 사용자 선택 + 디바이스 정보 표시 |
| 7 | DB 스키마 | PostgreSQL 기반 13개 테이블. 감사 로그 비정규화 + 불변성 트리거. Temporal 처리(인덱스 + 스케줄러 병행) |
| 8 | API 설계 | 5개 그룹(인증, 권한체크, 역할관리, 사용자-역할, 위임+승인). batch 권한 체크로 UI 성능 최적화 |
| 9 | OPA 연동 | Hybrid 패턴(Gateway 기본인증 + 서비스별 OPA Sidecar). 장애 Fallback: read=DB RBAC, write=기본거부 |
| 10 | Policy 캐시 | 결정 캐시 60초, 번들 동기화 30초, 데이터 5분 + 이벤트 기반 무효화 |
| 11 | UI/UX | 8개 핵심 화면: 역할관리, Permission Matrix, 내 권한, 역할비교, 요청 워크플로우, 위임, CSV Import, 대시보드 |
| 12 | 감사 로그 | 메인 DB 분리, 비동기 이벤트 수집. 3-tier 저장(Hot/Warm/Cold). 시간 블록 해시 무결성. SIEM 연동(Syslog/Webhook/Kafka) |
| 13 | 에러 코드 | 체계적 에러 코드: AUTH_xxx, PERM_xxx, PROJ_xxx, VAL_xxx, SYS_xxx |
| 14 | 이상 탐지 | Phase 1: 규칙 기반(6개 패턴), Phase 2: ML 기반 anomaly scoring |
