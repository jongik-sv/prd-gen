# Part 4: 마스터 데이터 관리(MDM) 심층 설계 요약

## 개요
- **주제**: MDM Hub 데이터 모델, API, 동기화, 캐싱, 거버넌스, 벌크 처리, 데이터 품질/AI
- **참여 전문가**: Andrew Ng, Andrej Karpathy (AI), Martin Kleppmann (DB), Martin Fowler (아키텍처), Robert C. Martin (SW), Kent Beck (PM), Jakob Nielsen, Bruce Schneier (UX/보안)

---

## 1. 마스터 데이터 모델 설계

- **3단계 계층 구조**: CodeGroup(대) -> CodeGroup(중) -> Code(소), 최대 3단계 제한
- **코드 범위**: Common(system) + Project 이원화
- **생명주기**: Draft -> Active -> Deprecated -> Archived (4단계 상태 머신)
- **다국어**: 코드 값은 하나, 표시 이름만 로케일별 관리 (LocalizedName)
- **버전 관리**: 모든 변경 시 CodeVersion 테이블에 이력 기록

---

## 2. MDM Hub API 설계 (5대 카테고리)

1. **Code Group 관리 API**: CRUD + 계층 트리 조회
2. **Code 관리 API**: CRUD + 생명주기 전환 + 버전 이력
3. **Lookup API (고성능)**: 읽기 전용, ETag/Cache-Control, 배치 조회, Change Feed
4. **Bulk API**: Excel/CSV 임포트(비동기, dryRun), 익스포트
5. **Search API**: 통합 검색, scope/type 필터
- 에러 코드: MDM_001~010 (10개 표준 에러 코드)

---

## 3. DB 스키마 (PostgreSQL 9개 + Outbox)

- code_groups, code_group_names, master_codes, master_code_names, master_code_attributes
- code_versions, code_dependencies, code_mappings, bulk_import_jobs, outbox_events
- **물리 삭제 금지** (Soft Delete), JSONB 확장성, 자동 updated_at 트리거

---

## 4. 데이터 동기화 (Hybrid 모델)

- **Push (Kafka)**: 인증/권한 관련, 시스템 상태 코드, 비활성화/Deprecated 알림
- **Pull (폴링)**: 국가/통화 코드, 프로젝트 한정 코드, 메타데이터 변경
- **Outbox 패턴**: DB 트랜잭션 + outbox_events 원자적 기록 -> 릴레이 -> Kafka 발행
- **Kafka 파티셔닝**: Partition Key = groupCode, 프로젝트별 컨슈머 그룹

---

## 5. 캐싱 전략 (2-Tier)

- **Tier 1 (로컬 캐시)**: 인메모리, TTL 5분 (인증 관련 1분)
- **Tier 2 (Redis)**: TTL 30분, 이벤트 기반 무효화
- **장애 폴백**: Redis 다운 -> 로컬 TTL 연장 -> DB 직접 조회 -> stale data 허용
- **Master SDK**: 모든 캐싱 로직 캡슐화, 개발자는 한 줄 호출만

---

## 6. 변경 거버넌스 워크플로우

- **승인 체인**: 요청자 -> 영향 분석 -> Data Owner -> (고영향 시) System Admin
- **긴급 변경**: Fast-track 승인 + 사후 감사 필수
- **에스컬레이션**: 48시간 미승인 -> 상위 관리자, 72시간 -> System Admin
- **상태 머신**: pending -> approved/rejected/cancelled (잘못된 전이 차단)

---

## 7. 벌크 처리 (5단계 비동기 파이프라인)

1. Upload -> 2. Validate -> 3. Queue(승인) -> 4. Process -> 5. Result
- 행 단위 에러 리포팅, 부분 성공 허용, 롤백 지원(import_job_id 태깅)
- Export: xlsx/csv, 10,000행 이상 비동기

---

## 8. 데이터 품질 관리 및 AI 적용

### 중복 탐지
- Levenshtein, Jaro-Winkler, Soundex/Metaphone (Phase 2)
- 임베딩 기반 의미적 중복 (Phase 4)

### 5대 품질 지표
- 완전성, 일관성, 적시성, 정확성, 유효성

### AI 로드맵
- **Phase 1**: CodeUsageLog 수집 시작 (AI의 전제조건)
- **Phase 2**: 규칙 기반 중복 탐지, 통계 기반 이상 탐지
- **Phase 3**: TF-IDF 자동 분류, 사용 패턴 기반 추천
- **Phase 4**: 임베딩 중복, LLM 설명 생성, 다국어 번역 제안

---

## 핵심 결정 사항 (26개)

| # | 항목 | 결정 |
|---|------|------|
| 1 | 계층 구조 | 최대 3단계, 추가 분류는 metadata로 |
| 2 | 코드 범위 | Common(system) + Project 이원화 |
| 3 | 생명주기 | 4단계 상태 머신 |
| 4 | 다국어 | 코드 값 단일, 표시 이름만 로케일별 |
| 5 | 버전 관리 | 모든 변경 시 버전 생성 |
| 6-8 | API | 4분류, ETag 캐싱, 표준 응답 포맷 |
| 9-10 | DB | 9개 테이블 + Outbox, 물리 삭제 금지 |
| 11-13 | 동기화 | Hybrid Push/Pull, Outbox 패턴, groupCode 파티셔닝 |
| 14-17 | 캐시 | 2-Tier(5분+30분), 이벤트 무효화, SDK 캡슐화, 장애 폴백 |
| 18-20 | 거버넌스 | Data Owner/Admin 승인, 긴급 Fast-track, 48h 에스컬레이션 |
| 21-22 | 벌크 | 5단계 파이프라인, 행 단위 에러 + 롤백 |
| 23-26 | 품질/AI | 중복 탐지(단계적), 5대 지표, Phase 1부터 수집, 커스텀 규칙 엔진 |
