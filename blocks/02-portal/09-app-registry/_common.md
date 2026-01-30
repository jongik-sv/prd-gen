---
category: app-registry
label: App Registry 전략
summary: Remote App의 등록 정보(manifest)를 관리하는 방식을 결정하여, 앱 추가·변경·삭제 시의 유연성을 조절한다.
context:
  - Shell App은 어떤 Remote App이 존재하는지, 각 앱의 진입점(entry)과 경로(routes)가 무엇인지 알아야 한다
  - 이 정보를 어디서 가져오느냐에 따라 앱 배포의 유연성과 안정성이 달라진다
  - 빌드 시 정적으로 포함하면 안정적이지만, 런타임에 동적으로 조회하면 Shell 재배포 없이 앱을 추가할 수 있다
dependencies:
  - 01-mf-integration
---

## 배경

App Registry는 포털에 등록된 모든 Remote App의 메타데이터를 관리하는 중앙 저장소이다. Shell App은 이 Registry를 통해 다음 정보를 얻는다:

### manifest 구조

각 Remote App은 다음 필드를 포함하는 manifest를 선언한다:

| 필드 | 설명 | 예시 |
|------|------|------|
| id | 앱 고유 식별자 | "project-a" |
| name | 사용자에게 표시되는 이름 | "프로젝트 A" |
| basePath | Shell 라우터의 기본 경로 | "/project-a" |
| entry | 앱 진입점 URL | "https://cdn.example.com/project-a/remoteEntry.js" |
| routes | 앱 내부 경로 목록 (이름, 아이콘, 그룹, 사이드바 표시 여부) | [{path: "/dashboard", name: "대시보드", ...}] |
| requiredPermissions | 접근에 필요한 권한 목록 | ["PROJECT_A_ACCESS"] |
| version | 앱 버전 | "1.2.0" |
| framework | 사용 프레임워크 | "react" |
| loadStrategy | 통합 방식 | "module-federation" |

### 선택지 요약

| 선택지 | 앱 추가 시 Shell 재배포 | 동적 갱신 | 오프라인 동작 |
|--------|----------------------|----------|-------------|
| 정적 manifest | 필요 | 불가 | 가능 (빌드에 포함) |
| 하이브리드 (정적 + 런타임 API) | 불필요 | 가능 | 가능 (정적 폴백) |
