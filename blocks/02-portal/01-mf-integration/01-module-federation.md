---
id: mf-module-federation
category: mf-integration
label: Module Federation
summary: Webpack 5 Module Federation으로 빌드 타임에 Remote App을 통합하여 디펜던시를 공유한다.
pros:
  - React 등 공유 디펜던시를 한 번만 로딩하여 번들 크기 대폭 절감
  - 같은 JS 컨텍스트에서 실행되므로 앱 간 통신이 빠르고 자연스러움
  - TypeScript 타입 공유가 용이하여 개발 생산성이 높음
  - Webpack 생태계의 HMR, 코드 스플리팅 등 기존 도구를 그대로 활용 가능
cons:
  - 모든 Remote App이 Webpack 5를 사용해야 하는 빌드 도구 종속
  - 공유 디펜던시 버전 충돌 시 런타임 오류 발생 가능 (singleton 설정 필요)
  - 한 앱의 전역 오류가 다른 앱에 전파될 수 있음 (격리 없음)
  - Vite, Rollup 등 다른 번들러를 사용하는 팀과 호환 불가
bestFor:
  - 전체 Remote App이 React + Webpack 5로 통일된 프로젝트
  - 단일 프론트엔드 팀이 모든 앱을 관리하는 구조
  - 초기 로딩 성능이 최우선인 내부 시스템
---

## 개요

Module Federation은 Webpack 5에 내장된 플러그인으로, 서로 다른 빌드 결과물(Remote) 간에 모듈을 런타임에 공유할 수 있게 한다. Shell App이 Host 역할을 하고, 각 Remote App이 자신의 모듈을 노출(expose)하면, Host가 런타임에 이를 가져와 마운트한다.

이 방식의 가장 큰 이점은 **디펜던시 공유**다. 예를 들어 모든 Remote App이 React 18을 사용한다면, React는 Shell App에서 한 번만 로딩되고 모든 Remote App이 이를 공유한다. 이로써 수백 KB의 중복 다운로드를 제거할 수 있다.

## 아키텍처

### Host-Remote 구조

```
Shell App (Host)
├── remoteEntry.js 로딩 (CDN)
├── 공유 디펜던시 제공 (react, react-dom, react-router)
└── Remote App mount
    ├── Remote App A (remoteEntry.js → expose ./App)
    ├── Remote App B (remoteEntry.js → expose ./App)
    └── Remote App C (remoteEntry.js → expose ./App)
```

### Webpack 설정 방식

**Shell App (Host) 설정:**
Shell App의 Webpack 설정에서는 ModuleFederationPlugin을 사용한다. 플러그인의 name 속성에 'shell'을 지정하고, remotes 속성에 각 Remote App의 이름과 remoteEntry.js의 CDN URL을 매핑한다. 런타임에 App Registry로부터 동적으로 주입하는 방식도 가능하다. shared 속성에는 react, react-dom, react-router-dom 등 공유할 디펜던시를 선언하되, singleton을 true로 설정하고 requiredVersion으로 허용 버전 범위(예: ^18.0.0)를 지정한다.

**Remote App 설정:**
Remote App의 Webpack 설정에서도 동일하게 ModuleFederationPlugin을 사용한다. name에 앱 고유 이름을 지정하고, filename을 'remoteEntry.js'로 설정한다. exposes 속성에서 외부에 노출할 모듈의 경로를 지정하며(예: './App' 키에 './src/bootstrap' 경로를 매핑), shared에는 Host와 동일하게 공유 디펜던시를 선언한다.

## 디펜던시 공유 전략

Module Federation의 핵심 가치인 디펜던시 공유는 세밀한 관리가 필요하다:

| 설정 | 설명 | 사용 시점 |
|------|------|----------|
| `singleton: true` | 전체 앱에서 단일 인스턴스만 허용 | React, React DOM 등 전역 상태가 있는 라이브러리 |
| `requiredVersion` | 허용 버전 범위 지정 | 메이저 버전 호환 보장 |
| `eager: true` | 초기 번들에 포함 (지연 로딩 안 함) | Shell App의 핵심 라이브러리 |
| `strictVersion: true` | 버전 불일치 시 런타임 에러 발생 | 버전 충돌을 명시적으로 감지하고 싶을 때 |

### 버전 충돌 시나리오

Remote App A가 React 18.2, Remote App B가 React 18.3을 요구하는 경우:
- `singleton: true` + `requiredVersion: '^18.0.0'` 설정이면 → 하나의 React 인스턴스만 로딩 (호스트 버전 우선)
- 메이저 버전이 다르면 (`^17` vs `^18`) → 런타임 경고 또는 에러 발생
- 이를 방지하려면 공유 디펜던시의 버전을 팀 간에 동기화하는 거버넌스가 필요

## 동적 Remote 로딩

빌드 타임에 모든 Remote를 지정하지 않고, 런타임에 App Registry로부터 Remote URL을 받아 동적으로 로딩할 수 있다. 동적 로딩 절차는 다음과 같다. 먼저 해당 Remote App의 remoteEntry.js 스크립트를 동적으로 페이지에 삽입한다. 이후 Webpack의 공유 스코프 초기화 함수(__webpack_init_sharing__)를 호출하여 'default' 스코프를 초기화하고, window 객체에서 해당 앱 이름으로 등록된 컨테이너를 가져와 공유 스코프로 초기화한다. 마지막으로 컨테이너에서 노출된 모듈(예: './App')을 가져와 팩토리 함수를 실행하면, RemoteApp 계약을 구현한 객체를 얻을 수 있다.

## 성능 특성

| 항목 | 수치/특징 |
|------|----------|
| 초기 로딩 | 공유 디펜던시로 인해 전체 번들 크기 30~50% 절감 |
| 앱 전환 | 같은 컨텍스트이므로 수십 ms 수준 |
| 메모리 | React 인스턴스를 공유하므로 탭 당 메모리 사용량 적음 |
| 빌드 시간 | 각 Remote App 독립 빌드 가능, Shell 재빌드 불필요 |

## 제약 사항 및 고려사항

1. **빌드 도구 종속**: Webpack 5가 필수. Vite를 쓰는 팀은 vite-plugin-federation을 사용할 수 있으나 안정성이 Webpack 대비 낮음
2. **격리 부재**: 모든 앱이 같은 window, document, global scope를 공유하므로, CSS 충돌과 전역 변수 오염에 주의해야 함
3. **타입 안전성**: Remote 모듈의 타입은 빌드 시점에 자동 공유되지 않으므로, 별도 타입 패키지(@portal/types)를 배포해야 함
4. **버전 관리**: 공유 디펜던시의 버전 동기화를 위한 팀 간 협의 프로세스 필요
