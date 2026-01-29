# Part 4: 마스터 데이터 관리(MDM) 심층 설계 브레인스토밍

> **시리즈**: 7-Expert Brainstorming for PRD/TRD Design (Part 4 of 7)
> **주제**: Master Data Management Hub -- 데이터 모델, API, 동기화, 캐싱, 거버넌스
> **일시**: 2026-01-29
> **형식**: 전문가 간 대화형 토의

---

## 참여 전문가

| # | 분야 | 인물 | 역할 |
|---|------|------|------|
| 1 | AI | Andrew Ng + Andrej Karpathy | AI 기반 데이터 품질 관리 |
| 2 | DB | Martin Kleppmann | 데이터 모델링, 동기화, 캐싱 |
| 3 | 아키텍처 | Martin Fowler | 시스템 아키텍처, 동기화 전략 |
| 4 | SW | Robert C. Martin (Uncle Bob) | API 설계, 클린 코드 |
| 5 | PM | Kent Beck (사회자) | 워크플로우, 벌크 처리 |
| 6 | UX/보안 | Jakob Nielsen + Bruce Schneier | UX 원칙, 보안 거버넌스 |

---

## 이전 세션 컨텍스트 요약

- MDM Hub는 독립 서비스로 운영, Single Source of Truth
- API 기반 참조, 로컬 캐시 + 서킷 브레이커
- 변경 거버넌스: 요청 -> 검토 -> 승인
- 코드 생명주기: Draft -> Active -> Deprecated -> Archived
- 다국어 코드명 지원 (locale 기반)
- 민감 필드 수준 접근 제어
- CDC 파이프라인 동기화, 최종 일관성(eventual consistency)

---

## 토의 4.1: 마스터 데이터 모델 설계

**Beck**: 자, Part 4를 시작하겠습니다. 이번 세션의 핵심은 MDM Hub의 구체적인 데이터 모델입니다. Kleppmann, 이전 세션에서 논의한 계층 구조를 구체화해 주시죠.

**Kleppmann**: 네, 코드 계층 구조부터 정리하겠습니다. 실무에서 가장 많이 보는 패턴은 3단계 계층입니다.

```
CodeGroup (대분류) -> CodeGroup (중분류) -> Code (소분류)
최대 깊이: 3
```

예를 들어 설명하면:

```
[대분류] SYSTEM_STATUS (시스템 상태 코드)
  ├── [중분류] USER_STATUS (사용자 상태)
  │     ├── [코드] ACTIVE   -> "활성"
  │     ├── [코드] INACTIVE -> "비활성"
  │     └── [코드] LOCKED   -> "잠김"
  └── [중분류] ORDER_STATUS (주문 상태)
        ├── [코드] PENDING  -> "대기중"
        ├── [코드] CONFIRMED-> "확인됨"
        └── [코드] SHIPPED  -> "발송됨"
```

**Fowler**: 깊이를 3으로 제한한 이유가 있나요?

**Kleppmann**: 실무 경험상 3단계를 넘어가면 탐색이 극도로 어려워지고, UI에서도 표현하기 힘들어집니다. 더 세분화가 필요하면 코드의 `metadata`나 `attributes`로 해결하는 것이 낫습니다. 무한 재귀를 허용하면 쿼리 성능도 급격히 떨어지고요.

**Uncle Bob**: 동의합니다. 제한 없는 깊이는 곧 제한 없는 복잡성입니다. 코드로도 깔끔하게 표현할 수 없게 됩니다.

**Kleppmann**: 그리고 코드의 범위(scope) 개념이 중요합니다. 두 가지로 나눕니다:

| 구분 | 범위 | 예시 |
|------|------|------|
| Common Codes | 시스템 전체 공유 | 국가코드, 통화코드, 상태코드 |
| Project Codes | 프로젝트 내 한정 | 프로젝트별 카테고리, 프로젝트별 등급 |

**Beck**: 생명주기도 정리해 주시죠.

**Kleppmann**: 코드의 생명주기는 명확한 상태 머신입니다:

```
Draft ──(activate)──> Active ──(deprecate)──> Deprecated ──(archive)──> Archived
  │                     │
  └──(delete)──> [삭제] └──(deactivate)──> Draft (예외적 롤백)
```

- **Draft**: 초안 상태. 아직 다른 시스템에서 참조 불가
- **Active**: 활성 상태. 조회 API에서 반환됨
- **Deprecated**: 사용 중단 예정. 기존 참조는 유효하지만 신규 사용 불가
- **Archived**: 아카이브 완료. 히스토리 조회만 가능

**Ng**: 다국어 지원은 어떻게 하나요?

**Kleppmann**: 코드 값(value)은 하나이고, 표시 이름(display name)만 로케일별로 관리합니다. 예를 들어:

```
Code: { code: "KR", value: "KR" }
  Names: [
    { locale: "ko", name: "대한민국" },
    { locale: "en", name: "South Korea" },
    { locale: "ja", name: "韓国" }
  ]
```

이렇게 하면 코드 값으로 시스템 간 통합할 때 일관성이 유지되고, 사용자에게 보여줄 때만 로케일에 맞게 표시하면 됩니다.

**Kleppmann**: 그리고 버저닝입니다. 모든 변경은 새로운 버전을 생성합니다. 코드 자체에 `version` 필드가 있고, 변경 이력은 별도 `CodeVersion` 테이블에 기록됩니다. 이는 거버넌스 감사 추적에 필수적입니다.

**Beck**: 이제 TypeScript 인터페이스로 정리해 봅시다.

**Kleppmann**: 전체 데이터 모델입니다:

```typescript
/**
 * 코드 그룹: 대분류/중분류 계층 구조
 * 최대 깊이 3단계 (대분류 -> 중분류 -> 소분류 코드)
 */
interface CodeGroup {
  id: string;                          // UUID
  parentId?: string;                   // 상위 그룹 ID (대분류는 null)
  code: string;                        // 고유 그룹 코드 (e.g., "COUNTRY", "USER_STATUS")
  depth: number;                       // 계층 깊이: 1(대분류), 2(중분류), 3(소분류)
  scope: 'system' | 'project';        // 시스템 공통 vs 프로젝트 한정
  projectId?: string;                  // scope='project'일 때만 사용
  status: 'active' | 'inactive';      // 그룹 활성 상태
  sortOrder: number;                   // 정렬 순서
  metadata: Record<string, any>;       // 확장 메타데이터 (JSON)
  names: LocalizedName[];              // 다국어 그룹명
  descriptions: LocalizedName[];       // 다국어 설명
}

/**
 * 마스터 코드: 실제 코드 값
 * CodeGroup 하위에 속함
 */
interface MasterCode {
  id: string;                          // UUID
  groupId: string;                     // 소속 그룹 ID
  code: string;                        // 그룹 내 고유 코드 (e.g., "KR", "ACTIVE")
  value: string;                       // 실제 값 (보통 code와 동일하지만, 다를 수 있음)
  status: 'draft' | 'active' | 'deprecated' | 'archived';  // 생명주기 상태
  sortOrder: number;                   // 정렬 순서
  effectiveFrom: Date;                 // 유효 시작일
  effectiveTo?: Date;                  // 유효 종료일 (null이면 무기한)
  version: number;                     // 현재 버전 번호
  metadata: Record<string, any>;       // 확장 메타데이터 (JSON)
  names: LocalizedName[];              // 다국어 코드명
  attributes: CodeAttribute[];         // 확장 속성 (유연한 key-value)
}

/**
 * 다국어 이름/설명
 */
interface LocalizedName {
  locale: string;                      // 'ko', 'en', 'ja', 'zh' 등
  name: string;                        // 해당 로케일의 표시명
  description?: string;                // 해당 로케일의 설명 (선택)
}

/**
 * 코드 확장 속성
 * 코드 그룹별로 다른 속성을 유연하게 추가 가능
 */
interface CodeAttribute {
  key: string;                         // 속성 키 (e.g., "icon", "color", "priority")
  value: any;                          // 속성 값
  type: 'string' | 'number' | 'boolean' | 'date';  // 값 타입
}

/**
 * 코드 변경 이력 (버전 관리)
 * 모든 변경은 새 버전 생성 + 승인 워크플로우
 */
interface CodeVersion {
  id: string;                          // UUID
  codeId: string;                      // 대상 코드 ID
  version: number;                     // 버전 번호
  changes: ChangeDetail[];             // 변경 상세 목록
  changedBy: string;                   // 변경 요청자
  changedAt: Date;                     // 변경 일시
  approvedBy?: string;                 // 승인자
  approvalStatus: 'pending' | 'approved' | 'rejected';  // 승인 상태
}

/**
 * 변경 상세 (필드 단위)
 */
interface ChangeDetail {
  field: string;                       // 변경 필드명
  before: any;                         // 변경 전 값
  after: any;                          // 변경 후 값
}
```

**Uncle Bob**: 인터페이스가 깔끔합니다. 한 가지 짚고 싶은 것은 `CodeAttribute`의 `value`가 `any` 타입인 부분인데, 런타임에서 `type` 필드에 따른 검증이 반드시 필요합니다.

**Kleppmann**: 맞습니다. 저장 시점에 `type`과 `value`의 정합성을 검증하는 밸리데이션 레이어를 반드시 넣어야 합니다. 예를 들어 `type: 'number'`인데 `value: "abc"`이면 거부해야죠.

**Schneier**: `metadata`가 `Record<string, any>`인데, 여기에 민감 정보가 들어갈 가능성은 없나요?

**Kleppmann**: 좋은 지적입니다. `metadata`에는 비즈니스 로직에 필요한 확장 정보만 넣도록 가이드하고, 민감 정보(개인정보 등)는 절대 포함하지 않도록 정책적으로 제한해야 합니다. 코드 그룹별로 허용되는 metadata 키를 스키마로 정의하는 방법도 고려할 수 있습니다.

**Beck**: 좋습니다. 모델이 명확하게 정리되었습니다. 다음으로 넘어가죠.

---

## 토의 4.2: MDM Hub 서비스 API 설계

**Beck**: Uncle Bob, API 설계를 이끌어 주시죠. 이전 세션에서 Contract-First 원칙을 강조하셨는데요.

**Uncle Bob**: 네, Contract-First입니다. 구현 전에 API 계약을 먼저 확정합니다. MDM Hub의 API는 크게 4가지 카테고리로 나눕니다:

1. **Code Group 관리 API** -- CRUD + 계층 조회
2. **Code 관리 API** -- CRUD + 생명주기 전환 + 버전 이력
3. **Lookup API** -- 고성능 조회 (캐시 최적화)
4. **Bulk API** -- 대량 처리 (임포트/익스포트)
5. **Search API** -- 통합 검색

이 분리가 중요한 이유는 각 카테고리의 비기능 요구사항이 완전히 다르기 때문입니다.

**Fowler**: 맞습니다. Lookup API는 초당 수천 건 호출이 예상되고, Bulk API는 한 건이지만 수만 행을 처리합니다. 이를 하나의 서비스로 묶되 내부적으로 리소스 격리가 필요합니다.

### 4.2.1 Code Group 관리 API

**Uncle Bob**: 먼저 코드 그룹 관리입니다:

```
# 그룹 목록 조회 (필터링 지원)
GET /master/groups
  Query Parameters:
    - scope: 'system' | 'project' (선택)
    - projectId: string (scope=project일 때)
    - status: 'active' | 'inactive' (선택)
    - parentId: string (특정 부모 하위만, 선택)
    - page: number (기본 1)
    - pageSize: number (기본 20, 최대 100)

# 특정 그룹 상세 조회
GET /master/groups/{groupId}

# 그룹 생성
POST /master/groups
  Body: CreateCodeGroupRequest

# 그룹 수정
PUT /master/groups/{groupId}
  Body: UpdateCodeGroupRequest

# 그룹 삭제 (소프트 삭제 -> status를 inactive로 변경)
DELETE /master/groups/{groupId}

# 그룹 전체 계층 트리 조회
GET /master/groups/{groupId}/tree
  Response: 해당 그룹 기준 하위 전체 트리 (코드 포함)
```

**Kleppmann**: `DELETE`가 소프트 삭제인 점이 중요합니다. 마스터 데이터는 물리적 삭제를 하면 안 됩니다. 기존에 이 코드를 참조하는 데이터가 있을 수 있으니까요.

**Uncle Bob**: 정확합니다. RESTful 관점에서 `DELETE`의 의미를 "리소스 비활성화"로 재정의하는 거죠. 이건 문서에 명확히 기술해야 합니다.

### 4.2.2 Code 관리 API

**Uncle Bob**: 다음, 코드 관리 API:

```
# 그룹 내 코드 목록 조회 (페이지네이션 + 필터)
GET /master/groups/{groupId}/codes
  Query Parameters:
    - status: 'draft' | 'active' | 'deprecated' | 'archived' (선택, 복수 가능)
    - locale: string (표시 언어, 기본 'ko')
    - page: number (기본 1)
    - pageSize: number (기본 20, 최대 200)
    - sort: string (e.g., 'sortOrder', 'code', 'updatedAt')
    - direction: 'asc' | 'desc'

# 특정 코드 상세 조회
GET /master/codes/{codeId}
  Query Parameters:
    - locale: string (선택)
    - includeVersions: boolean (기본 false)

# 코드 생성 (Draft 상태로 생성)
POST /master/groups/{groupId}/codes
  Body: CreateMasterCodeRequest

# 코드 수정 (새 버전 생성)
PUT /master/codes/{codeId}
  Body: UpdateMasterCodeRequest

# 코드 삭제 (Archived로 전환)
DELETE /master/codes/{codeId}

# 코드 버전 이력 조회
GET /master/codes/{codeId}/versions
  Query Parameters:
    - page: number
    - pageSize: number

# 코드 사용 중단 처리
POST /master/codes/{codeId}/deprecate
  Body: { reason: string, effectiveDate?: Date }

# 코드 활성화
POST /master/codes/{codeId}/activate
  Body: { effectiveFrom?: Date }
```

### 4.2.3 Lookup API (고성능, 캐시 최적화)

**Uncle Bob**: Lookup API는 가장 높은 성능이 요구됩니다. 읽기 전용이고, 캐시 친화적으로 설계합니다:

```
# 단일 코드 조회 (groupCode + codeValue로 직접 조회)
GET /master/lookup/{groupCode}/{codeValue}
  Query Parameters:
    - locale: string (기본 'ko')
  Headers:
    - If-None-Match: ETag (304 캐시 지원)
  Response Headers:
    - ETag: "version-hash"
    - Cache-Control: max-age=300

# 그룹 내 전체 코드 조회
GET /master/lookup/{groupCode}
  Query Parameters:
    - locale: string
    - status: 'active' (기본값, Lookup은 active만)
  Response Headers:
    - ETag: "group-version-hash"
    - Cache-Control: max-age=300

# 배치 조회 (여러 그룹/코드를 한 번에)
POST /master/lookup/batch
  Body: {
    requests: [
      { groupCode: "COUNTRY", codeValues: ["KR", "US", "JP"] },
      { groupCode: "CURRENCY" },  // 전체 조회
      { groupCode: "USER_STATUS", codeValues: ["ACTIVE"] }
    ],
    locale: "ko"
  }

# 변경 피드 (동기화용)
GET /master/lookup/changes?since={timestamp}
  Query Parameters:
    - since: ISO 8601 timestamp (필수)
    - groupCodes: string[] (선택, 특정 그룹만)
    - limit: number (기본 100, 최대 1000)
  Response: {
    changes: MasterDataEvent[],
    nextSince: string,  // 다음 폴링용 타임스탬프
    hasMore: boolean
  }
```

**Fowler**: Lookup API에서 `ETag`과 `Cache-Control` 헤더를 사용하는 것이 좋습니다. HTTP 수준 캐싱을 활용하면 CDN이나 API Gateway 레벨에서도 캐싱이 가능해집니다.

**Kleppmann**: 변경 피드 API의 `nextSince` 패턴은 Cursor-based pagination과 유사합니다. 클라이언트가 마지막으로 받은 타임스탬프를 기억하고, 다음 폴링 때 그 이후 변경분만 가져가는 거죠. 이벤트 유실 없이 안정적으로 동기화할 수 있습니다.

### 4.2.4 Bulk API

**Uncle Bob**: 대량 처리 API:

```
# 엑셀/CSV 업로드 (비동기)
POST /master/bulk/import
  Content-Type: multipart/form-data
  Body:
    - file: Excel/CSV 파일
    - groupId: string (대상 코드 그룹)
    - mode: 'create_only' | 'update_only' | 'upsert'
    - dryRun: boolean (검증만 수행, 실제 반영 안 함)
  Response: {
    jobId: string,
    status: 'queued',
    estimatedRows: number
  }

# 엑셀/CSV 다운로드
GET /master/bulk/export
  Query Parameters:
    - groupId: string (필수)
    - format: 'xlsx' | 'csv' (기본 'xlsx')
    - locale: string (기본 'ko')
    - includeInactive: boolean (기본 false)
  Response: Binary file download

# 비동기 작업 상태 조회
GET /master/bulk/jobs/{jobId}
  Response: BulkImportJobStatus
```

### 4.2.5 Search API

**Uncle Bob**: 마지막으로 통합 검색:

```
# 코드 및 그룹 통합 검색
GET /master/search
  Query Parameters:
    - q: string (검색어, 필수)
    - scope: 'system' | 'project' (선택)
    - projectId: string (선택)
    - type: 'code' | 'group' | 'all' (기본 'all')
    - locale: string (기본 'ko')
    - page: number
    - pageSize: number
```

### 4.2.6 표준 응답 포맷

**Uncle Bob**: 모든 API의 응답은 통일된 포맷을 따릅니다:

```typescript
/**
 * 표준 성공 응답 (단일 항목)
 */
interface SingleResponse<T> {
  status: 'success';
  data: T;
  timestamp: string;
}

/**
 * 표준 성공 응답 (목록, 페이지네이션)
 */
interface PaginatedResponse<T> {
  status: 'success';
  data: T[];
  pagination: {
    page: number;          // 현재 페이지 (1부터)
    pageSize: number;      // 페이지 크기
    totalItems: number;    // 전체 항목 수
    totalPages: number;    // 전체 페이지 수
  };
  timestamp: string;
}

/**
 * 표준 에러 응답
 */
interface ErrorResponse {
  status: 'error';
  error: {
    code: string;          // 에러 코드 (e.g., "MDM_001")
    message: string;       // 사람이 읽을 수 있는 메시지
    details?: any;         // 추가 상세 정보
    field?: string;        // 특정 필드 에러 시
  };
  timestamp: string;
}

/**
 * 벌크 작업 상태 응답
 */
interface BulkImportJobStatus {
  jobId: string;
  status: 'queued' | 'validating' | 'processing' | 'completed' | 'failed' | 'partial_success';
  totalRows: number;
  processedRows: number;
  successRows: number;
  errorRows: number;
  errors?: Array<{
    row: number;
    field: string;
    message: string;
    value: any;
  }>;
  startedAt?: Date;
  completedAt?: Date;
  resultFileUrl?: string;  // 에러 리포트 다운로드 URL
}
```

**Nielsen**: API 응답에서 `timestamp`를 항상 포함하는 것은 디버깅과 캐시 무효화에 매우 유용합니다. 프론트엔드에서도 "마지막 동기화: 3분 전" 같은 UX를 제공할 수 있고요.

**Uncle Bob**: 에러 코드도 체계적으로 정리해야 합니다:

| 에러 코드 | HTTP Status | 설명 |
|-----------|-------------|------|
| MDM_001 | 400 | 잘못된 요청 파라미터 |
| MDM_002 | 404 | 코드/그룹 없음 |
| MDM_003 | 409 | 중복 코드 |
| MDM_004 | 409 | 상태 전환 불가 (잘못된 생명주기 전환) |
| MDM_005 | 422 | 검증 실패 (필수 필드 누락 등) |
| MDM_006 | 403 | 권한 없음 |
| MDM_007 | 423 | 승인 대기 중 (수정 잠김) |
| MDM_008 | 429 | 요청 한도 초과 |
| MDM_009 | 500 | 내부 서버 오류 |
| MDM_010 | 422 | 의존성 위반 (참조 중인 코드 삭제 시도) |

**Beck**: 에러 코드 체계가 깔끔합니다. API 설계 완료. 다음은 DB 스키마로 가시죠.

---

## 토의 4.3: 마스터 DB 스키마 설계 (ERD)

**Beck**: Kleppmann, 지금까지 정의한 TypeScript 인터페이스를 실제 DB 스키마로 옮겨봅시다.

**Kleppmann**: 네, PostgreSQL 기준으로 DDL을 작성하겠습니다. 9개의 핵심 테이블과 인덱스, 제약 조건을 포함합니다.

### 4.3.1 전체 ERD 관계도

```
code_groups ──< code_group_names
    │
    ├── (self-ref: parent_id)
    │
    └──< master_codes ──< master_code_names
              │
              ├──< master_code_attributes
              │
              ├──< code_versions
              │
              ├──< code_dependencies (source)
              │
              └──< code_dependencies (target)

code_mappings (source_group ──> target_group 매핑)

bulk_import_jobs (독립)
```

### 4.3.2 SQL DDL

```sql
-- ============================================================
-- 1. code_groups: 코드 그룹 (대분류/중분류 계층)
-- ============================================================
CREATE TABLE code_groups (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id       UUID REFERENCES code_groups(id) ON DELETE RESTRICT,
    code            VARCHAR(100) NOT NULL,
    depth           SMALLINT NOT NULL CHECK (depth BETWEEN 1 AND 3),
    scope           VARCHAR(20) NOT NULL DEFAULT 'system' CHECK (scope IN ('system', 'project')),
    project_id      VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive')),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    metadata_json   JSONB DEFAULT '{}',
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    created_by      VARCHAR(200) NOT NULL,

    -- scope='project'이면 project_id 필수
    CONSTRAINT chk_project_scope CHECK (
        (scope = 'system' AND project_id IS NULL) OR
        (scope = 'project' AND project_id IS NOT NULL)
    )
);

-- 고유 제약: 같은 scope+project 내에서 code 유일
CREATE UNIQUE INDEX uq_code_groups_code
    ON code_groups (code, scope, COALESCE(project_id, '__SYSTEM__'));

-- 계층 조회 성능용
CREATE INDEX idx_code_groups_parent ON code_groups (parent_id);
CREATE INDEX idx_code_groups_scope ON code_groups (scope, project_id);
CREATE INDEX idx_code_groups_status ON code_groups (status);

-- ============================================================
-- 2. code_group_names: 코드 그룹 다국어 이름
-- ============================================================
CREATE TABLE code_group_names (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id    UUID NOT NULL REFERENCES code_groups(id) ON DELETE CASCADE,
    locale      VARCHAR(10) NOT NULL,
    name        VARCHAR(500) NOT NULL,
    description TEXT,

    CONSTRAINT uq_group_name_locale UNIQUE (group_id, locale)
);

CREATE INDEX idx_group_names_group ON code_group_names (group_id);
CREATE INDEX idx_group_names_locale ON code_group_names (locale);

-- ============================================================
-- 3. master_codes: 마스터 코드 (실제 코드 값)
-- ============================================================
CREATE TABLE master_codes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id        UUID NOT NULL REFERENCES code_groups(id) ON DELETE RESTRICT,
    code            VARCHAR(100) NOT NULL,
    value           VARCHAR(500) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'active', 'deprecated', 'archived')),
    sort_order      INTEGER NOT NULL DEFAULT 0,
    effective_from  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    effective_to    TIMESTAMP WITH TIME ZONE,
    version         INTEGER NOT NULL DEFAULT 1,
    metadata_json   JSONB DEFAULT '{}',
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    created_by      VARCHAR(200) NOT NULL,

    -- 유효 기간 검증
    CONSTRAINT chk_effective_period CHECK (
        effective_to IS NULL OR effective_to > effective_from
    )
);

-- 그룹 내 코드 유일 (같은 그룹에 같은 code 불가)
CREATE UNIQUE INDEX uq_master_codes_group_code
    ON master_codes (group_id, code);

-- Lookup 성능 핵심 인덱스
CREATE INDEX idx_master_codes_group ON master_codes (group_id);
CREATE INDEX idx_master_codes_status ON master_codes (status);
CREATE INDEX idx_master_codes_group_status ON master_codes (group_id, status);
CREATE INDEX idx_master_codes_value ON master_codes (group_id, value);

-- 유효 기간 범위 검색용
CREATE INDEX idx_master_codes_effective ON master_codes (effective_from, effective_to);

-- 변경 시간 기반 동기화 (Change Feed)
CREATE INDEX idx_master_codes_updated ON master_codes (updated_at);

-- ============================================================
-- 4. master_code_names: 마스터 코드 다국어 이름
-- ============================================================
CREATE TABLE master_code_names (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_id     UUID NOT NULL REFERENCES master_codes(id) ON DELETE CASCADE,
    locale      VARCHAR(10) NOT NULL,
    name        VARCHAR(500) NOT NULL,
    description TEXT,

    CONSTRAINT uq_code_name_locale UNIQUE (code_id, locale)
);

CREATE INDEX idx_code_names_code ON master_code_names (code_id);
CREATE INDEX idx_code_names_locale ON master_code_names (locale);

-- ============================================================
-- 5. master_code_attributes: 코드 확장 속성 (key-value)
-- ============================================================
CREATE TABLE master_code_attributes (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_id     UUID NOT NULL REFERENCES master_codes(id) ON DELETE CASCADE,
    attr_key    VARCHAR(200) NOT NULL,
    attr_value  TEXT,                      -- 문자열로 저장, 타입에 따라 변환
    attr_type   VARCHAR(20) NOT NULL DEFAULT 'string'
                    CHECK (attr_type IN ('string', 'number', 'boolean', 'date')),

    CONSTRAINT uq_code_attribute_key UNIQUE (code_id, attr_key)
);

CREATE INDEX idx_code_attrs_code ON master_code_attributes (code_id);
CREATE INDEX idx_code_attrs_key ON master_code_attributes (attr_key);

-- ============================================================
-- 6. code_versions: 코드 변경 이력 (감사 추적)
-- ============================================================
CREATE TABLE code_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code_id         UUID NOT NULL REFERENCES master_codes(id) ON DELETE RESTRICT,
    version         INTEGER NOT NULL,
    changes_json    JSONB NOT NULL,            -- ChangeDetail[] 배열
    changed_by      VARCHAR(200) NOT NULL,
    changed_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    approval_status VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (approval_status IN ('pending', 'approved', 'rejected')),
    approved_by     VARCHAR(200),
    approved_at     TIMESTAMP WITH TIME ZONE,
    comment         TEXT,

    CONSTRAINT uq_code_version UNIQUE (code_id, version)
);

CREATE INDEX idx_code_versions_code ON code_versions (code_id);
CREATE INDEX idx_code_versions_status ON code_versions (approval_status);
CREATE INDEX idx_code_versions_changed_at ON code_versions (changed_at);

-- ============================================================
-- 7. code_dependencies: 코드 간 의존 관계
-- ============================================================
CREATE TABLE code_dependencies (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_code_id    UUID NOT NULL REFERENCES master_codes(id) ON DELETE RESTRICT,
    target_code_id    UUID NOT NULL REFERENCES master_codes(id) ON DELETE RESTRICT,
    dependency_type   VARCHAR(50) NOT NULL,    -- 'requires', 'excludes', 'maps_to' 등
    description       TEXT,

    CONSTRAINT uq_code_dependency UNIQUE (source_code_id, target_code_id, dependency_type),
    CONSTRAINT chk_no_self_dependency CHECK (source_code_id != target_code_id)
);

CREATE INDEX idx_code_deps_source ON code_dependencies (source_code_id);
CREATE INDEX idx_code_deps_target ON code_dependencies (target_code_id);

-- ============================================================
-- 8. code_mappings: 프로젝트별 코드 매핑
-- ============================================================
CREATE TABLE code_mappings (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_group_id  UUID NOT NULL REFERENCES code_groups(id) ON DELETE RESTRICT,
    source_code      VARCHAR(100) NOT NULL,
    target_group_id  UUID NOT NULL REFERENCES code_groups(id) ON DELETE RESTRICT,
    target_code      VARCHAR(100) NOT NULL,
    project_id       VARCHAR(100),            -- null이면 전역 매핑
    description      TEXT,

    CONSTRAINT uq_code_mapping UNIQUE (source_group_id, source_code, target_group_id, COALESCE(project_id, '__GLOBAL__'))
);

CREATE INDEX idx_code_mappings_source ON code_mappings (source_group_id, source_code);
CREATE INDEX idx_code_mappings_target ON code_mappings (target_group_id, target_code);
CREATE INDEX idx_code_mappings_project ON code_mappings (project_id);

-- ============================================================
-- 9. bulk_import_jobs: 벌크 임포트 작업 관리
-- ============================================================
CREATE TABLE bulk_import_jobs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_name           VARCHAR(500) NOT NULL,
    group_id            UUID REFERENCES code_groups(id),
    import_mode         VARCHAR(20) NOT NULL DEFAULT 'upsert'
                            CHECK (import_mode IN ('create_only', 'update_only', 'upsert')),
    status              VARCHAR(20) NOT NULL DEFAULT 'queued'
                            CHECK (status IN ('queued', 'validating', 'processing', 'completed', 'failed', 'partial_success')),
    total_rows          INTEGER DEFAULT 0,
    success_rows        INTEGER DEFAULT 0,
    error_rows          INTEGER DEFAULT 0,
    error_details_json  JSONB DEFAULT '[]',    -- 행별 에러 상세
    created_by          VARCHAR(200) NOT NULL,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    started_at          TIMESTAMP WITH TIME ZONE,
    completed_at        TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_bulk_jobs_status ON bulk_import_jobs (status);
CREATE INDEX idx_bulk_jobs_created ON bulk_import_jobs (created_by, created_at DESC);

-- ============================================================
-- 공통: updated_at 자동 갱신 트리거
-- ============================================================
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_code_groups_updated
    BEFORE UPDATE ON code_groups
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_master_codes_updated
    BEFORE UPDATE ON master_codes
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

**Uncle Bob**: DDL이 깔끔합니다. `ON DELETE RESTRICT`를 대부분 사용한 것이 좋습니다. 마스터 데이터는 함부로 cascading delete 하면 안 되니까요.

**Kleppmann**: 맞습니다. 유일하게 `ON DELETE CASCADE`를 사용한 곳은 이름(names)과 속성(attributes) 테이블뿐입니다. 이건 부모 코드가 삭제되면 같이 삭제되어도 무방한 종속 데이터니까요. 하지만 `code_versions`는 `RESTRICT`입니다. 이력은 절대 유실되면 안 됩니다.

**Schneier**: `changes_json`에 들어가는 before/after 값에 민감 정보가 포함될 수 있습니다. 감사 로그 접근 권한을 별도로 관리해야 합니다.

**Kleppmann**: 좋은 지적입니다. `code_versions` 테이블에 대한 SELECT 권한을 일반 사용자에게는 부여하지 않고, 감사 역할(audit role)에만 허용하는 것이 좋겠습니다.

**Fowler**: `code_mappings` 테이블이 흥미롭습니다. 이건 어떤 용도인가요?

**Kleppmann**: 프로젝트마다 같은 개념을 다른 코드로 쓰는 경우가 있습니다. 예를 들어 A 프로젝트에서 "상태"를 `1, 2, 3`으로, B 프로젝트에서 `ACTIVE, INACTIVE, PENDING`으로 쓸 때, 이 매핑 테이블로 변환 관계를 정의합니다. 시스템 통합에서 필수적인 기능입니다.

**Beck**: ERD가 완성됐습니다. 9개 테이블, 관계, 인덱스, 제약 조건까지. 다음은 데이터 동기화로 넘어갑시다.

---

## 토의 4.4: 데이터 동기화 전략 (Push/Pull/Hybrid)

**Beck**: 마스터 데이터가 변경됐을 때, 이를 다른 서비스에 어떻게 전파할지 논의합시다. Fowler, Kleppmann, 이 주제를 이끌어 주시죠.

**Fowler**: 동기화 전략에는 크게 세 가지 모델이 있습니다:

| 모델 | 방식 | 장점 | 단점 |
|------|------|------|------|
| **Push** | 이벤트 발행 (Kafka/RabbitMQ) | 실시간, 낮은 지연 | 인프라 복잡, 순서 보장 어려움 |
| **Pull** | 클라이언트가 주기적 폴링 | 단순, 클라이언트 통제 | 지연 발생, 불필요한 트래픽 |
| **Hybrid** | 중요도에 따라 Push/Pull 선택 | 균형 잡힌 접근 | 두 가지 모두 구현 필요 |

**Kleppmann**: 저는 Hybrid를 강력히 추천합니다. 모든 마스터 데이터를 Push로 보내면 이벤트 폭풍이 발생할 수 있고, 모든 것을 Pull로 하면 중요한 변경이 지연됩니다.

**Fowler**: 동의합니다. 기준을 정해야 합니다:

### 4.4.1 Push 대상 (이벤트 기반)

```
[Push 대상 -- 즉시 전파 필요]
- 인증/권한 관련 코드 (역할, 권한 레벨)
- 시스템 상태 코드 (서비스 상태, 기능 플래그)
- 자주 사용되는 코드 그룹 (사전 지정)
- 코드 비활성화/Deprecated (기존 사용처에 즉시 알려야 함)
```

### 4.4.2 Pull 대상 (폴링/온디맨드)

```
[Pull 대상 -- 지연 허용]
- 국가/통화 코드 (거의 변하지 않음)
- 프로젝트 한정 코드 (해당 프로젝트만 관심)
- 메타데이터/속성 변경 (표시명 변경 등)
- 신규 코드 추가 (기존 시스템에 즉각 영향 없음)
```

### 4.4.3 이벤트 스키마

**Kleppmann**: Push 모델의 이벤트 스키마입니다:

```typescript
/**
 * 마스터 데이터 변경 이벤트
 * Kafka 메시지의 value 구조
 */
interface MasterDataEvent {
  eventId: string;                     // 이벤트 고유 ID (UUID, 멱등성 보장용)
  eventType: 'CODE_CREATED'            // 코드 신규 생성
            | 'CODE_UPDATED'           // 코드 수정
            | 'CODE_DEPRECATED'        // 코드 사용 중단
            | 'CODE_ARCHIVED'          // 코드 아카이브
            | 'GROUP_CHANGED';         // 그룹 변경
  timestamp: Date;                     // 이벤트 발생 시각
  groupCode: string;                   // 코드 그룹 코드
  codeValue?: string;                  // 변경된 코드 값 (그룹 변경 시 null)
  changeDetails: {
    before?: Record<string, any>;      // 변경 전 상태 (생성 시 null)
    after?: Record<string, any>;       // 변경 후 상태 (삭제 시 null)
  };
  changedBy: string;                   // 변경자
  projectScope?: string;               // 프로젝트 한정 코드인 경우 프로젝트 ID
}
```

### 4.4.4 Kafka 토픽 설계

**Kleppmann**: Kafka 토픽 구조는 이렇게 설계합니다:

```
[토픽 구조]
master-data.changes                    # 모든 변경을 담는 통합 토픽
master-data.{groupCode}.changes        # 그룹별 전용 토픽 (선택적)

[파티셔닝]
- Partition Key: groupCode
- 같은 그룹의 변경은 같은 파티션으로 → 순서 보장
- 그룹 간에는 순서 보장 불필요 (독립적)

[컨슈머 그룹]
- 프로젝트별 컨슈머 그룹: project-{projectId}-mdm-consumer
- 각 프로젝트가 독립적으로 오프셋 관리
- 한 프로젝트가 느려도 다른 프로젝트에 영향 없음
```

**Fowler**: 토픽을 두 단계로 나누는 이유를 설명해 주시겠어요?

**Kleppmann**: `master-data.changes`는 모든 변경이 흐르는 통합 토픽입니다. 모든 변경을 모니터링하고 싶은 서비스(예: 감사 로그, 데이터 품질 체크)가 구독합니다. `master-data.{groupCode}.changes`는 특정 그룹에만 관심 있는 서비스용입니다. 예를 들어 결제 서비스는 `master-data.CURRENCY.changes`만 구독하면 됩니다.

**Uncle Bob**: 이벤트 발행의 원자성은 어떻게 보장하나요? DB 변경과 이벤트 발행이 동시에 성공해야 하잖아요.

**Kleppmann**: Outbox 패턴을 씁니다. DB 트랜잭션 안에서 `master_codes` 테이블 변경과 함께 `outbox_events` 테이블에 이벤트를 기록합니다. 별도의 이벤트 릴레이 프로세스가 outbox를 읽어서 Kafka에 발행하고, 발행 완료 후 outbox에서 삭제합니다. 이렇게 하면 "DB는 변경됐는데 이벤트는 유실" 같은 문제가 없습니다.

```sql
-- Outbox 테이블 (이벤트 발행 보장용)
CREATE TABLE outbox_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,      -- 'MasterCode', 'CodeGroup'
    aggregate_id    UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    payload_json    JSONB NOT NULL,             -- MasterDataEvent JSON
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    published       BOOLEAN NOT NULL DEFAULT FALSE,
    published_at    TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_outbox_unpublished ON outbox_events (published, created_at)
    WHERE published = FALSE;
```

**Fowler**: Outbox 패턴은 Debezium 같은 CDC 도구로 자동화할 수도 있습니다. Debezium이 outbox 테이블의 변경을 감지해서 자동으로 Kafka에 발행해 주죠.

**Kleppmann**: 맞습니다. Debezium의 Outbox Event Router를 쓰면 별도 릴레이 프로세스 없이도 동작합니다. Phase 1에서는 간단한 폴링 기반 릴레이로 시작하고, Phase 2에서 Debezium으로 전환하는 것도 좋은 전략입니다.

**Beck**: Push/Pull Hybrid 전략과 Outbox 패턴까지 잘 정리됐습니다.

---

## 토의 4.5: 캐싱 전략 (2-tier, 무효화)

**Beck**: 마스터 데이터 조회는 모든 서비스에서 빈번하게 발생합니다. 캐싱 전략을 논의합시다.

**Kleppmann**: 2-tier 캐싱 전략을 제안합니다. 각 tier의 역할이 다릅니다:

```
[요청 흐름]
Application → Tier 1 (Local Cache) → Tier 2 (Redis) → PostgreSQL
                ↑ miss                  ↑ miss           ↑ miss
              5분 TTL               30분 TTL          DB 직접 조회
```

### 4.5.1 Tier 1: 애플리케이션 로컬 캐시

**Kleppmann**: 각 서비스 인스턴스 내부의 인메모리 캐시입니다:

```
- 구현: Node.js의 경우 node-cache 또는 Map + TTL 관리
- TTL: 비핵심 코드 5분, 인증 관련 코드 1분
- 최대 항목 수: 코드 그룹별 설정 가능 (기본 1000)
- 장점: 네트워크 호출 없음, 가장 빠름
- 단점: 인스턴스 간 정합성 없음 (최대 TTL 만큼 불일치 가능)
```

**Fowler**: 로컬 캐시의 정합성 문제는 마스터 데이터 특성상 대부분 허용 가능합니다. 코드명이 5분간 이전 버전으로 보여도 비즈니스에 치명적이지 않으니까요. 다만 인증/권한 관련 코드는 TTL을 1분으로 짧게 가져가는 것이 중요합니다.

### 4.5.2 Tier 2: 분산 캐시 (Redis)

**Kleppmann**: 모든 인스턴스가 공유하는 분산 캐시입니다:

```
- 구현: Redis Cluster
- TTL: 30분
- 키 포맷: master:{groupCode}:{codeValue}:{locale}
  예시: master:COUNTRY:KR:ko → { name: "대한민국", value: "KR", ... }
       master:COUNTRY:__ALL__:ko → [ 전체 국가 코드 목록 ]
- 무효화: 이벤트 기반 (마스터 데이터 변경 시 캐시 무효화 이벤트 발행)
```

**Fowler**: Redis 키 설계에서 `__ALL__` 패턴이 있는데, 그룹 전체 조회 결과도 캐시하는 건가요?

**Kleppmann**: 네, `GET /master/lookup/{groupCode}` 호출 시 그룹 전체 코드 목록을 하나의 키에 캐시합니다. 개별 코드 조회보다 그룹 전체 조회가 더 빈번한 패턴이거든요. 드롭다운 목록을 그릴 때 항상 전체 목록이 필요하니까요.

### 4.5.3 캐시 무효화 전략

**Kleppmann**: Cache-aside 패턴에 이벤트 기반 무효화를 결합합니다:

```
[캐시 무효화 흐름]
1. 마스터 데이터 변경 발생
2. DB 변경 + Outbox 이벤트 기록 (트랜잭션)
3. 이벤트 릴레이 → Kafka 발행
4. 캐시 무효화 컨슈머가 이벤트 수신
5. Redis에서 해당 키 삭제:
   - master:{groupCode}:{codeValue}:* (해당 코드의 모든 로케일)
   - master:{groupCode}:__ALL__:* (해당 그룹 전체 목록 캐시)
6. 로컬 캐시는 TTL 만료로 자연 무효화 (또는 Redis Pub/Sub으로 즉시 무효화)
```

**Fowler**: 로컬 캐시의 즉시 무효화가 필요한 경우가 있나요?

**Kleppmann**: 인증/권한 관련 코드라면 Redis Pub/Sub을 통해 모든 인스턴스에 무효화 신호를 보내는 것이 좋습니다. 하지만 일반 코드는 1-5분 내 TTL 만료로 충분합니다. 과도한 무효화 시그널은 오히려 성능을 해칩니다.

### 4.5.4 캐시 워밍

**Kleppmann**: 서비스 시작 시 자주 사용하는 코드 그룹을 미리 로드합니다:

```
[Cache Warming 전략]
- 서비스 시작 시 설정된 코드 그룹을 Redis에서 로컬 캐시로 로드
- Redis에 없으면 DB에서 읽어 Redis + 로컬 모두 적재
- 워밍 대상 그룹은 설정으로 관리 (preloadGroups)
- 워밍 실패 시 서비스 시작은 차단하지 않음 (graceful degradation)
```

### 4.5.5 장애 시 폴백

**Kleppmann**: 캐시 장애 시나리오별 대응:

```
[장애 시나리오]
1. Redis 다운:
   → 로컬 캐시 TTL 연장 (5분 → 30분)
   → 로컬 캐시 미스 시 DB 직접 조회
   → 서킷 브레이커로 Redis 호출 차단

2. Redis + 로컬 캐시 모두 미스:
   → DB 직접 조회 (폴백)
   → 결과를 로컬 캐시에만 저장 (Redis 복구 시까지)

3. DB도 다운:
   → 마지막 캐시된 데이터 사용 (stale data 허용)
   → 에러 로깅 + 알림
```

### 4.5.6 캐시 설정 인터페이스

```typescript
/**
 * MDM 캐시 설정
 * 각 서비스가 Master SDK 초기화 시 전달
 */
interface CacheConfig {
  localCache: {
    enabled: boolean;                  // 로컬 캐시 사용 여부
    maxEntries: number;                // 최대 캐시 항목 수 (기본 10000)
    ttlSeconds: number;                // TTL (기본 300 = 5분)
    criticalTtlSeconds: number;        // 인증/권한 관련 TTL (기본 60 = 1분)
    criticalGroups: string[];          // TTL 짧게 가져갈 그룹 코드 목록
  };
  distributedCache: {
    enabled: boolean;                  // Redis 캐시 사용 여부
    ttlSeconds: number;                // TTL (기본 1800 = 30분)
    keyPrefix: string;                 // 키 접두사 (기본 'master')
    connectionUrl: string;             // Redis 연결 URL
  };
  preloadGroups: string[];             // 서비스 시작 시 미리 로드할 그룹 코드
  fallback: {
    extendTtlOnRedisFailure: boolean;  // Redis 장애 시 로컬 TTL 연장 (기본 true)
    extendedTtlSeconds: number;        // 연장 TTL (기본 1800 = 30분)
    directDbFallback: boolean;         // 캐시 전체 미스 시 DB 직접 조회 (기본 true)
  };
}
```

**Uncle Bob**: 이 설정을 Master SDK가 투명하게 처리한다는 거죠? 서비스 개발자는 `masterSdk.getCode('COUNTRY', 'KR', 'ko')` 한 줄만 호출하면 되고, 내부적으로 로컬 → Redis → DB 순서로 조회하는 건가요?

**Kleppmann**: 정확합니다. 캐싱의 복잡성은 SDK 내부에 캡슐화됩니다. 서비스 개발자는 캐시 존재를 의식할 필요가 없습니다.

```typescript
// 서비스 개발자가 사용하는 코드 -- 캐싱은 투명
const countryName = await masterSdk.getCodeName('COUNTRY', 'KR', 'ko');
// → "대한민국" (내부적으로 로컬캐시 → Redis → DB 순서 조회)

const allCountries = await masterSdk.getCodesByGroup('COUNTRY', 'ko');
// → [{ code: "KR", name: "대한민국" }, { code: "US", name: "미국" }, ...]
```

**Nielsen**: 사용자 경험 관점에서 캐시 때문에 "방금 변경한 코드가 안 보여요" 같은 문의가 올 수 있습니다. 관리 화면에서 코드를 변경한 직후에는 해당 사용자의 캐시를 즉시 무효화하거나, 변경 후 "변경이 반영되기까지 최대 5분이 소요됩니다" 같은 안내가 필요합니다.

**Beck**: 좋은 UX 포인트입니다. 캐싱 전략 완료, 다음은 거버넌스입니다.

---

## 토의 4.6: 변경 거버넌스 워크플로우

**Beck**: 마스터 데이터 변경은 영향 범위가 크기 때문에 반드시 거버넌스가 필요합니다. 제가 Schneier와 함께 이 부분을 정리하겠습니다.

**Schneier**: 마스터 데이터는 보안 관점에서 "높은 무결성(High Integrity)" 자산입니다. 무단 변경이나 실수로 인한 변경이 전체 시스템에 파급 효과를 일으킬 수 있습니다. 따라서 모든 변경에 승인 프로세스가 필수입니다.

### 4.6.1 변경 요청 워크플로우

**Beck**: 기본 워크플로우입니다:

```
[변경 요청 흐름]
                                          ┌──────────────────┐
                                          │   System Admin   │
                                          │  (고영향 변경 시)  │
                                          └────────▲─────────┘
                                                   │ (에스컬레이션)
┌──────────┐    ┌────────────┐    ┌────────────────┐    ┌──────────┐
│ Requester│───>│  Impact    │───>│  Data Owner    │───>│  적용     │
│ (요청자)  │    │ Assessment │    │  (승인자)       │    │ (반영)   │
└──────────┘    │ (영향 분석) │    └────────────────┘    └──────────┘
                └────────────┘
```

### 4.6.2 변경 유형별 승인 요구사항

| 변경 유형 | 승인 주체 | 추가 조건 |
|-----------|-----------|-----------|
| 코드 추가 (신규) | Data Owner | - |
| 코드 수정 (이름/속성) | Data Owner | - |
| 코드 사용 중단 (Deprecate) | Data Owner | 의존 프로젝트에 알림 발송 |
| 코드 그룹 변경 | System Admin | 영향 범위 크므로 상위 승인 |
| 벌크 임포트 | System Admin | Dry-run 검증 결과 첨부 필수 |
| 긴급 변경 | Fast-track 승인 | 사후 감사(post-audit) 필수 |

**Schneier**: 긴급 변경(Emergency Change) 절차가 특히 중요합니다. 장애 상황에서 마스터 데이터를 빠르게 수정해야 할 때, 정상 승인 프로세스를 기다릴 수 없으니까요. 하지만 긴급 변경은 반드시 사후 감사를 받아야 합니다.

### 4.6.3 영향 분석 (Impact Assessment)

**Beck**: 변경 요청이 들어오면 시스템이 자동으로 영향 분석을 수행합니다:

```
[자동 영향 분석 항목]
1. 의존 프로젝트 파악: code_dependencies 테이블 조회
2. 의존 코드 파악: 해당 코드를 참조하는 다른 코드들
3. 영향 받는 사용자 수 추정: 해당 코드 그룹의 API 호출 통계 기반
4. 리스크 수준 자동 산정:
   - Low: 신규 추가, 이름 변경
   - Medium: 값 변경, 속성 수정
   - High: 사용 중단, 삭제, 그룹 구조 변경
```

### 4.6.4 알림 및 에스컬레이션

**Beck**: 승인 요청 시 알림과 에스컬레이션 정책:

```
[알림]
- 승인 요청 시: 이메일 + 포털 내 알림 (승인자에게)
- 승인/거부 시: 이메일 + 포털 내 알림 (요청자에게)
- 사용 중단(Deprecate) 시: 의존 프로젝트 담당자에게 알림

[에스컬레이션]
- 48시간 내 미승인 시: 자동 에스컬레이션 (상위 관리자에게)
- 72시간 내 미승인 시: System Admin에게 에스컬레이션
- 에스컬레이션 시 원 승인자에게도 리마인더 발송
```

### 4.6.5 거버넌스 TypeScript 인터페이스

```typescript
/**
 * 변경 요청
 */
interface ChangeRequest {
  id: string;                          // UUID
  type: 'create' | 'update' | 'deprecate' | 'archive' | 'bulk_import';
  targetType: 'code' | 'code_group';   // 변경 대상 종류
  targetId?: string;                   // 변경 대상 ID (신규 생성 시 null)
  changes: Record<string, {            // 필드별 변경 내역
    before: any;
    after: any;
  }>;
  requestedBy: string;                 // 요청자 ID
  requestedAt: Date;                   // 요청 일시
  status: 'pending' | 'approved' | 'rejected' | 'cancelled';
  approvalChain: ApprovalStep[];       // 승인 체인
  impactAnalysis?: ImpactAnalysis;     // 자동 영향 분석 결과
  comment?: string;                    // 요청 사유
  isEmergency: boolean;                // 긴급 변경 여부
}

/**
 * 승인 단계
 */
interface ApprovalStep {
  stepOrder: number;                   // 승인 순서
  approverRole: 'data_owner' | 'system_admin';
  approverId?: string;                 // 지정 승인자 (null이면 역할 기반)
  status: 'pending' | 'approved' | 'rejected' | 'skipped';
  decidedAt?: Date;
  decidedBy?: string;
  comment?: string;
}

/**
 * 영향 분석 결과
 */
interface ImpactAnalysis {
  dependentProjects: string[];         // 영향 받는 프로젝트 ID 목록
  dependentCodes: string[];            // 의존하는 코드 ID 목록
  affectedUsers: number;               // 영향 받는 추정 사용자 수
  riskLevel: 'low' | 'medium' | 'high';  // 리스크 수준
  details: string;                     // 분석 상세 설명
  analyzedAt: Date;                    // 분석 시점
}

/**
 * 알림
 */
interface GovernanceNotification {
  id: string;
  changeRequestId: string;
  recipientId: string;
  type: 'approval_request' | 'approval_result' | 'deprecation_notice' | 'escalation';
  channel: 'email' | 'portal' | 'both';
  title: string;
  message: string;
  sentAt: Date;
  readAt?: Date;
}
```

**Schneier**: 한 가지 더 추가할 것은, 모든 승인/거부 이력은 불변(immutable) 로그로 남겨야 합니다. 누가 언제 어떤 변경을 승인했는지는 컴플라이언스 감사에서 반드시 필요합니다.

**Uncle Bob**: 코드 수준에서 보면, 변경 요청 상태 전이도 명확한 상태 머신으로 구현해야 합니다:

```
pending ──(approve)──> approved
pending ──(reject)───> rejected
pending ──(cancel)───> cancelled
approved ──(apply)───> [변경 적용]
```

잘못된 전이(예: rejected → approved)는 코드 레벨에서 차단해야 합니다.

**Beck**: 거버넌스 워크플로우 정리 완료. 다음은 벌크 처리입니다.

---

## 토의 4.7: 벌크 처리 설계 (Excel Import/Export)

**Beck**: 마스터 데이터 초기 구축이나 대량 수정 시 벌크 처리가 필수입니다. Uncle Bob과 제가 이 부분을 정리하겠습니다.

**Uncle Bob**: 벌크 처리의 핵심 원칙은 "안전한 대량 처리"입니다. 한 줄의 오류가 전체를 망치면 안 됩니다.

### 4.7.1 Excel 템플릿

**Beck**: 코드 그룹별로 맞춤 엑셀 템플릿을 제공합니다:

```
[템플릿 구조 예시 -- COUNTRY 그룹]
┌─────────┬──────────┬─────────┬──────────┬──────────┬────────────┬────────────┐
│ code(*) │ value(*) │ name_ko │ name_en  │ name_ja  │ sort_order │ attr:icon  │
├─────────┼──────────┼─────────┼──────────┼──────────┼────────────┼────────────┤
│ KR      │ KR       │ 대한민국 │ S.Korea  │ 韓国     │ 1          │ 🇰🇷        │
│ US      │ US       │ 미국    │ USA      │ アメリカ  │ 2          │ 🇺🇸        │
│ JP      │ JP       │ 일본    │ Japan    │ 日本     │ 3          │ 🇯🇵        │
└─────────┴──────────┴─────────┴──────────┴──────────┴────────────┴────────────┘

(*) 필수 필드
- 코드 그룹의 지원 로케일에 따라 name_xx 컬럼 자동 생성
- 코드 그룹의 속성 정의에 따라 attr:xxx 컬럼 자동 생성
- 첫 번째 행: 헤더 (컬럼명)
- 두 번째 행: 데이터 타입/필수 여부 안내 (숨김 행)
- 세 번째 행부터: 실제 데이터
```

### 4.7.2 업로드 처리 파이프라인

**Uncle Bob**: 업로드는 5단계 비동기 파이프라인으로 처리합니다:

```
[비동기 처리 파이프라인]

1. Upload (업로드)
   └── 파일 수신, 기본 포맷 검증 (파일 형식, 크기)
   └── 즉시 jobId 반환 → 클라이언트는 폴링으로 진행 상황 확인

2. Validate (검증)
   └── 헤더 검증: 필수 컬럼 존재 여부
   └── 행별 검증:
       - 필수 필드 누락 체크
       - 데이터 타입 검증 (code 형식, 날짜 형식 등)
       - 코드 중복 체크 (파일 내 + DB 기존 데이터)
       - 참조 무결성 체크 (groupId 존재 여부 등)
   └── dryRun=true이면 여기서 종료, 검증 결과만 반환

3. Queue (대기열)
   └── 검증 통과 → 처리 큐에 등록
   └── 거버넌스: System Admin 승인 필요 (bulk_import 유형)
   └── 승인 완료 후 Processing으로 전환

4. Process (처리)
   └── 행 단위 처리 (트랜잭션)
   └── 성공/실패 카운트 기록
   └── 진행률 실시간 업데이트 (폴링으로 확인 가능)

5. Result (결과)
   └── 완료/부분 성공/실패 상태 기록
   └── 에러 상세 리포트 생성 (다운로드 가능)
   └── 변경 이벤트 발행 (성공한 건에 대해)
```

### 4.7.3 에러 처리

**Uncle Bob**: 에러 처리가 벌크 임포트의 핵심입니다:

```
[에러 처리 정책]

1. 행 단위 에러 리포팅:
   - 각 행의 성공/실패를 개별 추적
   - 실패 행에 대해 행 번호 + 필드 + 에러 메시지 기록

2. 부분 성공 허용 (사용자 확인 후):
   - 검증 단계에서 에러 행 목록 제공
   - 사용자가 "에러 행 제외하고 나머지 진행" 선택 가능
   - 또는 "전체 중단" 선택 가능

3. 롤백 지원:
   - 벌크 임포트로 생성된 코드들에 import_job_id 태깅
   - 문제 발생 시 해당 job의 변경 사항 일괄 롤백 가능

4. 에러 리포트:
   - 원본 엑셀 파일에 에러 컬럼 추가하여 다운로드 제공
   - 에러 행만 추출한 별도 파일도 제공
```

```typescript
/**
 * 벌크 임포트 에러 상세
 */
interface BulkImportError {
  row: number;              // 엑셀 행 번호
  field: string;            // 에러 발생 필드
  value: any;               // 입력된 값
  errorCode: string;        // 에러 코드
  message: string;          // 에러 메시지
}

/**
 * 벌크 임포트 검증 결과 (dryRun 응답)
 */
interface BulkValidationResult {
  jobId: string;
  totalRows: number;
  validRows: number;
  errorRows: number;
  errors: BulkImportError[];
  warnings: BulkImportError[];   // 경고 (진행 가능하지만 주의 필요)
  canProceed: boolean;           // 진행 가능 여부
  requiresApproval: boolean;     // 승인 필요 여부
}
```

### 4.7.4 Export (다운로드)

**Beck**: Export는 상대적으로 단순합니다:

```
[Export 기능]
- 코드 그룹 단위 다운로드
- 포맷: Excel (.xlsx) 또는 CSV
- 필터 옵션:
  - status 필터 (active만 vs 전체)
  - locale 선택 (어떤 언어로 이름 내보낼지)
  - 컬럼 선택 (필요한 컬럼만)
- 대용량 (10,000행 이상) 시 비동기 처리
- Content-Disposition 헤더로 파일명 지정
```

### 4.7.5 Job 모니터링 UI

**Nielsen**: UX 관점에서 벌크 처리 모니터링 화면에 필요한 요소:

```
[모니터링 UI 요소]
- 프로그레스 바: 처리 진행률 (processedRows / totalRows)
- 상태 배지: queued | validating | processing | completed | failed | partial_success
- 경과 시간 / 예상 완료 시간
- 실시간 카운터: 성공 N건 / 실패 N건 / 남은 N건
- 에러 목록 미리보기: 첫 5건 에러 표시
- 에러 리포트 다운로드 버튼 (처리 완료 후)
- 롤백 버튼 (부분 성공 시)
- 최근 작업 이력 목록
```

**Beck**: 벌크 처리 설계 완료. 마지막으로 데이터 품질 관리입니다.

---

## 토의 4.8: 데이터 품질 관리 (중복 탐지, AI 적용)

**Beck**: 마지막 토의입니다. Ng, Karpathy, 데이터 품질 관리와 AI 적용에 대해 논의해 주시죠.

**Ng**: 마스터 데이터의 품질은 전체 시스템 품질의 기반입니다. "Garbage in, garbage out"이죠. 먼저 전통적인 데이터 품질 기법부터 시작하고, 점진적으로 AI를 도입하는 로드맵을 제안합니다.

### 4.8.1 중복 탐지 (Duplicate Detection)

**Karpathy**: 마스터 코드에서 중복은 크게 두 가지입니다:

```
[중복 유형]
1. 정확한 중복 (Exact Match):
   - 같은 그룹 내 동일 코드값 → 유니크 제약으로 DB 레벨에서 방지

2. 유사 중복 (Fuzzy Match):
   - 다른 코드이지만 의미가 같은 경우
   - 예: "대한민국" vs "한국" vs "Korea" vs "South Korea"
   - 예: "ACTIVE" vs "ACT" vs "활성"
```

**Ng**: 유사 중복 탐지에는 여러 알고리즘을 조합합니다:

```
[유사도 알고리즘]

1. Levenshtein Distance (편집 거리):
   - 문자열 간 최소 편집 횟수
   - "ACTIVE" vs "ACTVE" → 거리 1 (유사)
   - 임계값: 거리 2 이하면 경고

2. Jaro-Winkler Similarity:
   - 접두사 가중치 부여, 0~1 사이 값
   - "COUNTRY" vs "CONTRY" → 0.96 (매우 유사)
   - 임계값: 0.85 이상이면 경고

3. Soundex / Metaphone (음성 유사도):
   - 발음이 비슷한 코드 탐지
   - 영어 코드에 효과적

4. 다국어 이름 비교:
   - 로케일별 이름 간 교차 비교
   - 한글 자모 분리 비교 (ㅎㅏㄴㄱㅜㄱ vs 대한민국)
```

**Karpathy**: Phase 4에서 AI를 도입하면 더 정교한 의미적 중복 탐지가 가능합니다:

```
[AI 기반 중복 탐지 -- Phase 4]
- 임베딩 기반 유사도: 코드명을 벡터로 변환 후 코사인 유사도 계산
- "대한민국" 임베딩과 "한국" 임베딩은 높은 유사도
- 다국어 임베딩 모델 (multilingual-e5, mBERT) 활용
- 새 코드 등록 시 기존 코드와 유사도 자동 체크
```

### 4.8.2 이상 탐지 (Anomaly Detection)

**Ng**: 비정상적인 패턴을 감지합니다:

```
[이상 탐지 규칙]

1. 급격한 대량 변경:
   - 단시간에 50건 이상 변경 → 경고
   - 정상 패턴 학습 후 임계값 자동 조정 (Phase 4)

2. 비정상적인 코드 패턴:
   - 코드 그룹의 기존 패턴과 다른 형식
   - 예: COUNTRY 그룹에서 기존 코드가 2글자(KR, US)인데 새 코드가 "KOREA"
   - 규칙 기반으로 시작, AI 기반으로 발전

3. 사용하지 않는 코드 탐지:
   - Lookup API 호출 로그 분석
   - 90일간 조회 0건인 코드 → 정리 후보 제안

4. 순환 의존성:
   - code_dependencies에서 A→B→C→A 같은 순환 탐지
   - 그래프 탐색 알고리즘으로 실시간 방지
```

### 4.8.3 자동 분류 (Auto-Classification)

**Karpathy**: 새 코드 등록 시 적절한 코드 그룹을 자동 제안합니다:

```
[자동 분류 -- Phase 4]

1. 학습 데이터:
   - 기존 코드의 (이름, 속성, 그룹) 쌍으로 분류 모델 학습

2. 제안 방식:
   - 사용자가 코드명 입력 → 상위 3개 그룹 추천 (신뢰도 %)
   - "대한민국" 입력 → COUNTRY (95%), REGION (3%), ORG (2%)

3. 모델:
   - 초기: TF-IDF + SVM (단순, 빠름)
   - 고도화: 파인튜닝된 BERT 분류기
```

### 4.8.4 데이터 품질 대시보드

**Ng**: 데이터 품질을 정량적으로 측정하고 모니터링합니다:

```
[품질 지표 (Metrics)]

1. 완전성 (Completeness):
   - 필수 필드 채움률
   - 다국어 이름 등록률 (ko: 100%, en: 85%, ja: 60%)
   - 목표: 필수 필드 100%, 다국어 90% 이상

2. 일관성 (Consistency):
   - 코드 형식 일관성 (대문자/소문자 혼용 등)
   - 속성 값 범위 이탈 건수
   - 목표: 형식 위반 0건

3. 적시성 (Timeliness):
   - 마지막 리뷰 이후 경과 시간
   - Deprecated 상태 유지 기간 (너무 오래되면 Archive 필요)
   - 목표: 6개월 이상 미리뷰 코드 0건

4. 정확성 (Accuracy):
   - 중복 코드 비율
   - 사용자 피드백 기반 오류 보고 건수
   - 목표: 중복 비율 1% 이하

5. 유효성 (Validity):
   - 유효 기간 만료 코드 중 Active 상태인 건수
   - 참조 무결성 위반 건수
   - 목표: 위반 0건
```

```typescript
/**
 * 데이터 품질 대시보드 요약
 */
interface DataQualityDashboard {
  timestamp: Date;
  overview: {
    totalGroups: number;
    totalCodes: number;
    activeCodesPercentage: number;
    overallQualityScore: number;        // 0~100
  };
  completeness: {
    requiredFieldCompletionRate: number; // 0~1
    localeCompletionRates: Record<string, number>; // locale별 등록률
    missingNames: Array<{ codeId: string; locale: string }>;
  };
  consistency: {
    formatViolations: number;
    namingConventionViolations: number;
    details: Array<{ codeId: string; issue: string }>;
  };
  timeliness: {
    staleCodesCount: number;            // 장기간 미리뷰
    overdueDeprecations: number;        // Deprecated 유지 기간 초과
    avgTimeSinceLastReview: number;     // 일 단위
  };
  duplicates: {
    suspectedDuplicates: number;
    duplicatePairs: Array<{
      codeId1: string;
      codeId2: string;
      similarity: number;
      algorithm: string;
    }>;
  };
}
```

### 4.8.5 품질 규칙 엔진

**Ng**: 코드 그룹별로 커스텀 품질 규칙을 설정할 수 있습니다:

```typescript
/**
 * 품질 규칙 정의
 * 코드 그룹별로 다른 규칙 적용 가능
 */
interface QualityRule {
  id: string;
  groupId?: string;                    // null이면 전체 적용
  name: string;                        // 규칙 이름
  type: 'format' | 'range' | 'required' | 'unique' | 'custom';
  severity: 'error' | 'warning' | 'info';
  config: {
    field: string;                     // 적용 대상 필드
    pattern?: string;                  // 정규식 패턴 (format 타입)
    min?: number;                      // 최소값 (range 타입)
    max?: number;                      // 최대값 (range 타입)
    customValidator?: string;          // 커스텀 검증 함수명 (custom 타입)
  };
  enabled: boolean;
  message: string;                     // 위반 시 메시지
}

// 사용 예시
const countryCodeRules: QualityRule[] = [
  {
    id: 'rule-001',
    groupId: 'group-country',
    name: '국가 코드 형식',
    type: 'format',
    severity: 'error',
    config: {
      field: 'code',
      pattern: '^[A-Z]{2}$'           // ISO 3166-1 alpha-2 형식
    },
    enabled: true,
    message: '국가 코드는 대문자 2자리여야 합니다 (ISO 3166-1 alpha-2)'
  },
  {
    id: 'rule-002',
    groupId: 'group-country',
    name: '한국어 이름 필수',
    type: 'required',
    severity: 'error',
    config: {
      field: 'names.ko'
    },
    enabled: true,
    message: '한국어 이름은 필수입니다'
  }
];
```

### 4.8.6 AI 적용 로드맵

**Ng**: AI 기능은 Phase 4에서 본격 도입하지만, 데이터 수집은 Phase 1부터 시작합니다:

```
[AI 적용 단계]

Phase 1 (데이터 수집):
  - Lookup API 호출 로그 수집 (어떤 코드가 자주 사용되는지)
  - 코드 변경 이력 수집 (패턴 학습용)
  - 사용자 행동 로그 수집 (검색어, 선택 패턴)
  - ★ 데이터가 없으면 AI는 무용지물이므로, Phase 1부터 수집 설계가 핵심

Phase 2 (규칙 기반):
  - Levenshtein/Jaro-Winkler 기반 중복 탐지
  - 통계 기반 이상 탐지 (평균 ± 3σ)
  - 규칙 기반 품질 스코어 계산

Phase 3 (ML 기초):
  - TF-IDF 기반 자동 분류
  - 사용 패턴 기반 코드 추천
  - 시계열 분석 기반 이상 탐지

Phase 4 (고급 AI):
  - 임베딩 기반 의미적 중복 탐지
  - LLM 기반 코드 설명 자동 생성
  - 다국어 번역 제안 (코드명 자동 번역)
  - 예측 분석: 코드 사용 추세 예측
```

**Karpathy**: Phase 1에서 데이터 수집 설계를 미리 해놓는 것이 정말 중요합니다. 나중에 AI를 도입하려고 할 때 데이터가 없으면 6개월 이상 추가로 기다려야 합니다. 로그 스키마를 미리 정의하고, 수집 파이프라인을 구축해 놓으면 Phase 4에서 바로 활용할 수 있습니다.

```typescript
/**
 * 코드 사용 로그 (Phase 1부터 수집)
 * AI 학습 데이터로 활용
 */
interface CodeUsageLog {
  timestamp: Date;
  groupCode: string;
  codeValue: string;
  action: 'lookup' | 'search' | 'select';  // 조회/검색/선택
  userId?: string;                     // 익명화된 사용자 ID
  projectId?: string;
  source: string;                      // 호출 서비스명
  searchQuery?: string;                // 검색이었다면 검색어
  responseTimeMs: number;              // 응답 시간
}
```

**Ng**: 이 로그 데이터로 할 수 있는 분석이 많습니다:

```
[Phase 4 AI 활용 사례]
1. "이 코드를 조회한 사용자는 이 코드도 함께 조회했습니다" → 관련 코드 추천
2. "최근 3개월간 이 코드 조회가 70% 감소" → Deprecation 후보 자동 제안
3. "이 검색어로 코드를 찾지 못한 경우가 많습니다" → 코드명 개선 제안
4. "신규 코드 '활성 상태'는 기존 코드 'ACTIVE'와 의미적으로 유사합니다" → 중복 경고
```

**Beck**: AI 로드맵까지 훌륭하게 정리됐습니다. 이제 전체 핵심 결정 사항을 요약하겠습니다.

---

## 핵심 결정 사항 요약

| # | 결정 항목 | 결정 내용 | 근거 | 담당 전문가 |
|---|-----------|-----------|------|-------------|
| 1 | 코드 계층 구조 | CodeGroup (대분류) -> CodeGroup (중분류) -> Code (소분류), **최대 3단계** | 3단계 초과 시 탐색 복잡도 급증, 쿼리 성능 저하. 추가 분류는 metadata/attributes로 해결 | Kleppmann |
| 2 | 코드 범위(Scope) | Common (system) + Project 이원화 | 시스템 전체 공통 코드와 프로젝트 한정 코드의 관리 주체와 생명주기가 다름 | Kleppmann |
| 3 | 코드 생명주기 | Draft -> Active -> Deprecated -> Archived (4단계 상태 머신) | 사용 중단(Deprecated)과 아카이브(Archived)를 분리하여 기존 참조 호환성 유지 | Kleppmann |
| 4 | 다국어 전략 | 코드 값은 하나, 표시 이름만 로케일별 관리 (LocalizedName) | 시스템 통합 시 코드 값의 일관성 보장, 표시만 국제화 | Kleppmann |
| 5 | 버전 관리 | 모든 변경 시 새 버전 생성, CodeVersion 테이블에 이력 기록 | 감사 추적, 변경 이력 추적, 롤백 가능성 확보 | Kleppmann |
| 6 | API 카테고리 분리 | 관리 API / Lookup API / Bulk API / Search API 4분류 | 비기능 요구사항(성능, 인증, 처리 방식)이 카테고리별로 상이 | Uncle Bob |
| 7 | Lookup API 캐싱 | ETag + Cache-Control 헤더, HTTP 레벨 캐싱 지원 | API Gateway/CDN 레벨 캐싱 활용, 클라이언트 304 Not Modified 지원 | Uncle Bob, Fowler |
| 8 | 응답 포맷 | 단일/목록/에러 3종 표준 응답 포맷 + 에러 코드 체계 (MDM_001~010) | API 일관성, 클라이언트 에러 핸들링 표준화 | Uncle Bob |
| 9 | DB 스키마 | PostgreSQL 9개 테이블 + Outbox 테이블 | 정규화된 스키마, JSONB로 확장성 확보, Outbox 패턴으로 이벤트 발행 원자성 보장 | Kleppmann |
| 10 | 삭제 정책 | 물리 삭제 금지 (Soft Delete), ON DELETE RESTRICT | 마스터 데이터 참조 무결성 보호, 이력 보존 | Kleppmann |
| 11 | 동기화 전략 | **Hybrid**: 인증/권한 관련 Push (Kafka), 일반 코드 Pull (Change Feed API) | 모든 코드를 Push하면 이벤트 폭풍, 모든 코드를 Pull하면 중요 변경 지연 | Fowler, Kleppmann |
| 12 | 이벤트 발행 원자성 | Outbox 패턴 + 이벤트 릴레이 (Phase 2에서 Debezium 전환 검토) | DB 변경과 이벤트 발행의 원자성 보장 | Kleppmann |
| 13 | Kafka 파티셔닝 | Partition Key = groupCode, 프로젝트별 컨슈머 그룹 | 같은 그룹 내 순서 보장, 프로젝트 간 독립성 | Kleppmann |
| 14 | 캐싱 구조 | 2-Tier: 로컬 캐시 (5분 TTL) + Redis (30분 TTL) | 로컬 캐시로 네트워크 비용 제거, Redis로 인스턴스 간 공유 | Kleppmann, Fowler |
| 15 | 캐시 무효화 | 이벤트 기반 Redis 무효화 + TTL 기반 로컬 캐시 자연 만료 | 실시간성과 단순성의 균형. 인증 관련만 Pub/Sub 즉시 무효화 | Kleppmann |
| 16 | 캐시 장애 폴백 | Redis 다운 시 로컬 TTL 연장 -> DB 직접 조회 -> stale data 허용 | 가용성 우선 (AP 모델). 마스터 데이터는 잠시 오래된 데이터가 허용됨 | Kleppmann |
| 17 | 캐시 투명성 | Master SDK가 모든 캐싱 로직 캡슐화 | 서비스 개발자 부담 최소화, 일관된 캐싱 동작 보장 | Uncle Bob |
| 18 | 거버넌스 워크플로우 | Requester -> Data Owner -> (고영향 시) System Admin 승인 체인 | 변경 영향도에 따른 적절한 승인 수준, 과도한 절차 방지 | Beck, Schneier |
| 19 | 긴급 변경 | Fast-track 승인 + 사후 감사 필수 | 장애 대응 신속성 확보, 사후 추적으로 보안 보완 | Schneier |
| 20 | 에스컬레이션 | 48시간 미승인 시 자동 에스컬레이션 | 변경 요청 적체 방지, SLA 준수 | Beck |
| 21 | 벌크 임포트 | 5단계 비동기 파이프라인 (Upload -> Validate -> Queue -> Process -> Result) | 대용량 안전 처리, 행 단위 에러 핸들링, 부분 성공 허용 | Beck, Uncle Bob |
| 22 | 벌크 에러 처리 | 행 단위 에러 리포팅 + 부분 성공 허용 (사용자 확인 후) + 롤백 지원 | 한 행의 에러가 전체 실패로 이어지지 않도록 | Uncle Bob |
| 23 | 중복 탐지 | Levenshtein + Jaro-Winkler (Phase 2), 임베딩 기반 의미적 중복 (Phase 4) | 단계적 고도화: 규칙 기반 -> ML 기반 | Ng, Karpathy |
| 24 | 데이터 품질 지표 | 완전성, 일관성, 적시성, 정확성, 유효성 5개 축 측정 | ISO 8000 데이터 품질 표준 기반 | Ng |
| 25 | AI 데이터 수집 | Phase 1부터 CodeUsageLog 수집 시작 | Phase 4 AI 도입 시 학습 데이터 확보 (6개월 이상 축적 필요) | Ng, Karpathy |
| 26 | 품질 규칙 엔진 | 코드 그룹별 커스텀 QualityRule 설정 가능 | 그룹마다 코드 형식과 규칙이 다르므로 유연한 규칙 엔진 필요 | Ng |

---

> **다음 세션 예고**: Part 5에서는 프론트엔드 아키텍처와 UI 컴포넌트 설계를 다룹니다.
