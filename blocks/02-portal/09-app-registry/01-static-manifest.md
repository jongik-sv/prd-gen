---
id: reg-static-manifest
category: app-registry
label: 빌드 시 정적 manifest
summary: Shell App 빌드 시 모든 Remote App의 manifest를 정적으로 포함하여, 런타임 API 의존 없이 안정적으로 동작한다.
pros:
  - 별도 Registry 서비스 없이 Shell 빌드에 manifest가 포함되어 가장 안정적
  - Registry API 장애에 영향받지 않아 가용성이 높음
  - 빌드 시점에 모든 앱 정보가 확정되어 타입 체크, 유효성 검증 가능
  - 초기 로딩 시 네트워크 요청 없이 즉시 메뉴/라우팅 구성 가능
cons:
  - 새 Remote App을 추가하거나 기존 앱의 경로를 변경하면 Shell App을 재빌드·재배포해야 함
  - Shell과 Remote App의 배포 주기가 결합됨 (Shell이 병목)
  - 앱 수가 많아지면 manifest 관리가 수동이고 번거로워짐
  - A/B 테스트, 점진적 롤아웃 등 동적 앱 구성이 불가
bestFor:
  - Remote App 수가 적고(5개 이하) 변경이 드문 안정적인 포털
  - Registry 서비스를 운영할 인프라가 없는 소규모 프로젝트
  - 최대 안정성과 가용성이 요구되는 환경 (오프라인 지원 포함)
---

## 개요

정적 manifest 전략은 Shell App의 빌드 과정에서 모든 Remote App의 manifest 정보를 번들에 포함하는 방식이다. 별도의 Registry API 없이, Shell App 코드 자체가 앱 목록과 라우팅 정보를 알고 있다.

이 방식은 **"빌드 타임 결정"** 원칙을 따른다. Shell이 빌드될 때 어떤 앱이 존재하고, 각 앱의 진입점이 어디이며, 어떤 경로를 사용하는지가 모두 확정된다.

## 구성 방식

### manifest 파일 위치

Shell App 프로젝트 내에 manifest 설정 파일을 두고, 빌드 시 이를 번들에 포함한다:

```
shell-app/
├── src/
│   ├── config/
│   │   └── apps-manifest.json   ← 모든 Remote App의 manifest
│   ├── router/
│   │   └── index.ts             ← manifest 기반 라우팅 구성
│   └── ...
└── ...
```

### manifest 내용 예시

manifest 파일에는 각 Remote App의 등록 정보를 배열로 정의한다. 각 항목은 앱 ID, 표시명, 기본 경로(basePath), 진입점 URL(entry), 내부 경로 목록(routes), 필요 권한(requiredPermissions), 버전, 프레임워크, 통합 방식(loadStrategy) 등을 포함한다.

사이드바 메뉴, 라우팅 테이블, 권한 검증 로직이 모두 이 manifest를 참조하여 구성된다.

## 빌드 프로세스

### 앱 추가/변경 시 프로세스

```
1. Remote App 개발·배포 (CDN에 빌드 결과물 업로드)
    │
2. Shell App의 apps-manifest.json에 새 앱 정보 추가/수정
    │
3. Shell App 재빌드
    │
4. Shell App 재배포 (CDN 또는 서버)
    │
5. 사용자가 새로고침 시 새 메뉴 표시
```

### 배포 커플링

- Remote App의 entry URL이 변경되면 Shell manifest도 수정 필요
- 여러 Remote App이 동시에 업데이트되면 Shell의 manifest를 한 번에 수정 후 배포
- Shell 배포 실패 시 Remote App 변경도 반영되지 않음

## 환경별 manifest

개발(dev), 스테이징(staging), 운영(production) 환경별로 다른 manifest를 사용할 수 있다:

| 환경 | manifest 파일 | entry URL 예시 |
|------|-------------|---------------|
| 개발 | apps-manifest.dev.json | https://dev-cdn.example.com/app-a/remoteEntry.js |
| 스테이징 | apps-manifest.staging.json | https://staging-cdn.example.com/app-a/remoteEntry.js |
| 운영 | apps-manifest.prod.json | https://cdn.example.com/app-a/remoteEntry.js |

빌드 시 환경 변수에 따라 적절한 manifest 파일을 선택한다.

## 유효성 검증

빌드 시점에 manifest의 유효성을 검증할 수 있다:

| 검증 항목 | 방법 |
|----------|------|
| ID 중복 검사 | 각 앱의 id가 고유한지 확인 |
| basePath 충돌 검사 | 경로가 겹치는 앱이 없는지 확인 |
| 필수 필드 검증 | id, name, basePath, entry 등 필수 필드 존재 여부 |
| URL 형식 검증 | entry URL이 유효한 형식인지 확인 |
| 권한 참조 검증 | requiredPermissions의 권한이 실제 존재하는지 확인 (옵션) |

## 제약 사항 및 고려사항

1. **배포 결합**: Remote App을 추가할 때마다 Shell을 재빌드해야 하므로, 빈번한 앱 변경 시 병목
2. **확장성**: 앱 수가 20개 이상으로 늘어나면 manifest 파일 수동 관리가 부담
3. **동적 기능 불가**: 특정 사용자에게만 앱을 노출하거나, A/B 테스트로 앱을 분기하는 등의 동적 구성 불가
4. **CDN 캐시**: Shell 재배포 시 CDN 캐시 무효화가 필요하며, 사용자가 새로고침하지 않으면 이전 manifest가 유지됨
