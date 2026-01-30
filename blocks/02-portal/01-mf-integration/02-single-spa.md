---
id: mf-single-spa
category: mf-integration
label: Single-SPA
summary: Single-SPA 프레임워크로 React, Vue, Angular 등 이기종 프레임워크 앱을 런타임에 독립 통합한다.
pros:
  - 프레임워크에 완전히 독립적이어서 React, Vue, Angular, Svelte 등 자유롭게 혼용 가능
  - 각 Remote App이 독립적인 빌드 파이프라인을 유지 (Webpack, Vite 등 무관)
  - 성숙한 생태계와 풍부한 문서, 커뮤니티 지원
  - 앱별 독립 배포 시 Shell App 재빌드가 불필요
cons:
  - 디펜던시 공유가 기본적으로 안 되므로, 각 앱이 React를 각각 로딩하여 번들 크기 증가
  - 공유 디펜던시를 Import Map으로 관리할 수 있으나 설정 복잡도가 높음
  - 여러 프레임워크가 동시에 로딩되면 메모리 사용량이 증가
  - 앱 간 상태 공유가 Module Federation 대비 불편 (별도 이벤트 버스 필요)
bestFor:
  - 다수의 개발팀이 서로 다른 프레임워크를 사용하는 대규모 조직
  - 레거시 앱(Angular, jQuery 등)을 점진적으로 통합해야 하는 마이그레이션 프로젝트
  - 빌드 도구를 통일하기 어려운 환경
---

## 개요

Single-SPA는 "메타 프레임워크"로서, 여러 SPA(Single Page Application)를 하나의 페이지에서 공존하게 한다. 각 앱은 자신만의 프레임워크와 빌드 도구를 사용하며, Single-SPA가 URL 라우팅에 따라 앱의 생명주기(bootstrap → mount → unmount)를 관리한다.

Module Federation이 "빌드 타임에 모듈을 공유"하는 접근이라면, Single-SPA는 "런타임에 독립 앱을 오케스트레이션"하는 접근이다. 이 차이로 인해 프레임워크 독립성이 높지만, 디펜던시 공유 효율은 낮다.

## 아키텍처

### 구성 요소

```
Single-SPA Root Config (Shell App)
├── Import Map (CDN URL → 모듈 매핑)
├── Layout Engine (라우트 → 앱 매핑)
└── 등록된 Applications
    ├── @portal/project-a (React 18 + Vite)
    ├── @portal/project-b (Vue 3 + Webpack)
    └── @portal/legacy-app (Angular 14)
```

### Root Config 동작 방식

Shell App의 엔트리 파일(root-config)에서 single-spa 라이브러리의 registerApplication 함수를 사용하여 각 Remote App을 등록한다. 등록 시 앱의 이름(예: '@portal/project-a'), SystemJS를 통한 앱 로딩 함수, 해당 앱이 활성화될 URL 경로(예: '/project-a'), 그리고 Shell이 전달하는 커스텀 속성(인증 토큰, 이벤트 버스 등)을 지정한다. 모든 앱 등록이 완료되면 start 함수를 호출하여 Single-SPA 라우터를 시작한다.

### Remote App 래퍼 구조

각 Remote App은 해당 프레임워크용 Single-SPA 어댑터를 사용하여 생명주기를 노출한다. 예를 들어 React 앱의 경우 single-spa-react 어댑터를 사용하여 React, ReactDOMClient, 루트 컴포넌트, 에러 바운더리를 설정한 뒤, 반환된 생명주기 객체에서 bootstrap, mount, unmount를 각각 export 한다. 이 export된 생명주기가 Shell의 RemoteApp 계약과 매핑된다.

## Import Map을 통한 모듈 관리

Single-SPA는 SystemJS와 Import Map을 사용하여 각 앱의 진입점을 관리한다. HTML 페이지에 systemjs-importmap 타입의 스크립트 태그를 선언하여, 각 앱의 모듈 이름과 CDN URL을 매핑한다. 예를 들어 '@portal/root-config', '@portal/project-a', '@portal/project-b' 같은 모듈 이름을 각각의 CDN 빌드 결과물 URL에 연결하고, react, react-dom 같은 공유 라이브러리도 CDN URL로 지정할 수 있다.

Import Map을 서버에서 동적으로 생성하면 App Registry와 연동하여 런타임에 앱 목록을 갱신할 수 있다.

## 디펜던시 공유 전략

Single-SPA 자체는 디펜던시 공유 기능이 없지만, Import Map으로 우회할 수 있다:

| 전략 | 방법 | 효과 |
|------|------|------|
| Import Map 공유 | React를 CDN URL로 지정, 모든 앱이 같은 URL 참조 | 브라우저 캐시로 중복 다운로드 방지 |
| Externals 설정 | 각 앱에서 react를 external로 지정 | 번들에서 제외 |
| Utility Module | 공통 유틸리티를 별도 패키지로 배포 | 코드 재사용 |

단, 이 방식은 Module Federation의 자동 공유 대비 설정이 복잡하고, 각 팀이 동일하게 external 설정을 해야 하는 협의 비용이 있다.

## 프레임워크별 어댑터

Single-SPA는 주요 프레임워크별 공식 어댑터를 제공한다:

| 프레임워크 | 어댑터 패키지 | 비고 |
|-----------|-------------|------|
| React | `single-spa-react` | React 16+ 지원 |
| Vue | `single-spa-vue` | Vue 2/3 지원 |
| Angular | `single-spa-angular` | Angular 9+ 지원, Zone.js 격리 필요 |
| Svelte | `single-spa-svelte` | Svelte 3+ |
| Vanilla JS | 직접 구현 | bootstrap/mount/unmount 직접 작성 |

## 성능 특성

| 항목 | 수치/특징 |
|------|----------|
| 초기 로딩 | Import Map 기반 병렬 로딩, 디펜던시 미공유 시 번들 크기 증가 |
| 앱 전환 | JS 모듈 로딩 후 mount → 수백 ms 수준 (프리페칭 시 개선 가능) |
| 메모리 | 프레임워크별 런타임이 각각 로딩되므로 Module Federation 대비 높음 |
| 빌드 시간 | 완전 독립 빌드, 팀 간 의존성 없음 |

## 제약 사항 및 고려사항

1. **SystemJS 종속**: Import Map 기반 동적 로딩을 위해 SystemJS가 필요하며, ES Module 네이티브 방식으로 전환 시 추가 설정이 필요
2. **CSS 격리 없음**: 앱 간 CSS가 같은 document에 삽입되므로, CSS Modules 또는 CSS-in-JS로 스코핑 필요
3. **디버깅 복잡도**: 여러 프레임워크가 동시에 실행되므로, DevTools에서 디버깅이 복잡해질 수 있음
4. **학습 곡선**: Root Config, Import Map, SystemJS, 프레임워크별 어댑터 등 이해해야 할 개념이 많음
