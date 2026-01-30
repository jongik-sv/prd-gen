---
id: reg-hybrid
category: app-registry
label: 하이브리드 (정적 + 런타임 API)
summary: Shell 빌드 시 기본 manifest를 포함하되, 런타임에 Registry API로 최신 정보를 갱신하여 Shell 재배포 없이 앱을 추가/변경한다.
pros:
  - Shell 재배포 없이 Remote App을 동적으로 추가·수정·삭제 가능
  - Registry API 장애 시 빌드 내장 manifest로 폴백하여 가용성 유지
  - A/B 테스트, 점진적 롤아웃, 사용자별 앱 노출 등 동적 구성 가능
  - Shell과 Remote App의 배포 주기를 완전히 분리하여 독립 배포 달성
cons:
  - Registry API 서버 구축·운영 필요
  - 초기 로딩 시 API 호출이 추가되어 약간의 지연 발생
  - 정적 manifest와 API 응답 간의 병합(merge) 로직 필요
  - Registry 데이터의 관리 UI가 추가로 필요
bestFor:
  - Remote App이 자주 추가/변경되는 대규모 포털
  - Shell과 Remote App의 배포 독립성이 중요한 마이크로 프론트엔드 환경
  - 사용자/역할별로 다른 앱 목록을 동적으로 구성해야 하는 플랫폼
---

## 개요

하이브리드 전략은 **빌드 타임 안정성**과 **런타임 유연성**을 동시에 달성한다. Shell App은 빌드 시 기본 manifest를 내장하여 항상 동작하도록 보장하고, 런타임에 Registry API를 호출하여 최신 앱 정보로 갱신한다.

이 이중 보험 구조로 인해 Registry API가 다운되어도 Shell은 빌드 내장 manifest로 동작하며, API가 정상이면 최신 정보를 반영한다.

## 아키텍처

### 로딩 시퀀스

```
Shell App 로딩
    │
    ├── 1. 빌드 내장 manifest로 즉시 메뉴/라우팅 구성 (빠른 초기 렌더링)
    │
    ├── 2. Registry API 비동기 호출 (백그라운드)
    │   │
    │   ├── 성공: API 응답으로 manifest 병합/갱신
    │   │   └── 새로 추가된 앱의 메뉴 표시, 삭제된 앱 숨기기
    │   │
    │   └── 실패: 빌드 내장 manifest 유지 (폴백)
    │       └── 콘솔에 경고 로그, 사용자에게는 영향 없음
    │
    └── 3. 캐싱: API 응답을 LocalStorage에 저장
        └── 다음 로딩 시 캐시 먼저 사용 → API로 갱신 (Stale-While-Revalidate)
```

### 병합(Merge) 전략

빌드 내장 manifest와 API 응답을 어떻게 합칠 것인지:

| 전략 | 동작 | 적합 상황 |
|------|------|----------|
| **API 우선 (Override)** | API 응답이 있으면 전체 교체 | API가 완전한 manifest를 반환할 때 |
| **병합 (Merge)** | 같은 ID의 앱은 API 버전이 우선, 없는 앱은 빌드 버전 유지 | API가 변경분만 반환할 때 |
| **추가만 (Additive)** | 빌드 manifest 유지 + API에서 새 앱만 추가 | 빌드 manifest가 기본, API는 확장만 |

권장: **API 우선** 방식으로, API가 전체 manifest를 반환하되 실패 시 빌드 내장 manifest로 폴백.

## Registry API 설계

### 주요 엔드포인트

| 엔드포인트 | 메서드 | 설명 |
|-----------|--------|------|
| /api/registry/apps | GET | 현재 사용자가 접근 가능한 전체 앱 목록 |
| /api/registry/apps/{id} | GET | 특정 앱의 manifest 상세 |
| /api/registry/apps | POST | 새 앱 등록 (관리자) |
| /api/registry/apps/{id} | PUT | 앱 정보 수정 (관리자) |
| /api/registry/apps/{id} | DELETE | 앱 삭제 (관리자) |
| /api/registry/health | GET | Registry 서비스 상태 확인 |

### 사용자별 앱 필터링

Registry API는 현재 인증된 사용자의 역할·권한에 따라 접근 가능한 앱만 반환한다. 이를 통해:

- 권한 없는 앱은 메뉴에 아예 표시되지 않음
- 사용자별/역할별 맞춤 메뉴 구성 가능
- 베타 앱을 특정 사용자 그룹에게만 노출하는 점진적 롤아웃 가능

## 캐싱 전략

API 호출의 지연과 실패를 완화하기 위한 다중 캐싱:

| 캐시 계층 | 위치 | 수명 | 용도 |
|----------|------|------|------|
| 빌드 내장 manifest | 코드 번들 | Shell 재빌드 시 갱신 | 최종 폴백 |
| LocalStorage 캐시 | 브라우저 | 마지막 API 성공 응답 | 빠른 초기 렌더링 |
| HTTP 캐시 | 브라우저/CDN | Cache-Control 헤더 기반 | API 응답 캐싱 |
| 메모리 캐시 | Shell App | 세션 동안 | 반복 조회 방지 |

Stale-While-Revalidate 패턴: 캐시된 데이터로 즉시 렌더링 → 백그라운드에서 API 호출 → 새 데이터 도착 시 갱신.

## Registry 관리 UI

관리자가 앱을 등록·수정하는 관리 화면이 필요하다:

| 기능 | 설명 |
|------|------|
| 앱 목록 조회 | 등록된 모든 앱의 상태, 버전, 활성 여부 |
| 앱 등록/수정 | manifest 필드 편집 (ID, 이름, 경로, entry URL, 권한 등) |
| 앱 활성/비활성 | 앱을 제거하지 않고 비활성화하여 메뉴에서 숨기기 |
| 배포 상태 확인 | 각 앱의 entry URL이 실제로 접근 가능한지 헬스체크 |
| 이력 조회 | 앱 등록/수정/삭제 이력 (누가, 언제, 무엇을) |

## 제약 사항 및 고려사항

1. **API 가용성**: Registry API가 장기간 다운되면 빌드 내장 manifest가 오래된 정보를 유지할 수 있음
2. **일관성**: API 응답이 캐시와 다를 때 메뉴가 깜빡이거나 순간적으로 앱 목록이 변경될 수 있음
3. **보안**: Registry API 자체가 해킹되면 악의적 entry URL이 배포될 수 있음 → API 접근 제어 및 URL 화이트리스트 필수
4. **마이그레이션**: 정적 manifest에서 하이브리드로 전환할 때 기존 manifest를 Registry DB로 이전하는 작업 필요
5. **관리 비용**: Registry 서비스 운영 + 관리 UI 개발·유지 비용
