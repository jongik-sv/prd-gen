---
id: mf-hybrid
category: mf-integration
label: Hybrid Adapter 패턴
summary: Contract 기반 Adapter 레이어로 Module Federation, Single-SPA, iframe을 모두 수용하여 통합 방식을 앱별로 선택한다.
pros:
  - 앱별로 최적의 통합 방식을 선택할 수 있는 유연성
  - 내부 React 앱은 Module Federation으로, 외부 앱은 iframe으로 등 혼합 운영 가능
  - Shell App 코드 변경 없이 새로운 통합 방식을 Adapter로 추가할 수 있음
  - 단계적 마이그레이션에 적합 (iframe → Single-SPA → Module Federation)
cons:
  - Adapter 레이어 자체의 설계/구현/유지보수 복잡도 증가
  - 통합 방식마다 다른 성능 특성을 가지므로, 전체 시스템의 성능 예측이 어려움
  - 팀 간 어떤 방식을 쓸지 거버넌스 결정 필요
  - 모든 방식의 인프라(Webpack MF 설정, Single-SPA 설정, iframe 통신 등)를 유지해야 함
bestFor:
  - 내부 앱과 외부 앱이 혼재된 대규모 엔터프라이즈 포털
  - 레거시 앱을 점진적으로 현대화하면서 동시에 새 앱도 개발하는 프로젝트
  - 통합 방식 결정을 미루고 싶은 초기 아키텍처 설계 단계
---

## 개요

Hybrid Adapter 패턴은 Shell App이 특정 통합 기술에 종속되지 않도록, Contract(계약)과 Adapter(어댑터) 레이어를 분리하는 설계다. Shell App은 `RemoteApp` 인터페이스만 알면 되고, 실제 로딩/마운트 방식은 Adapter가 처리한다.

이 접근의 핵심 가치는 **결정의 지연(Deferred Decision)**이 아닌, **결정의 다양화(Diversified Decision)**다. 프로젝트 초기에 단일 방식을 강제하는 대신, 각 Remote App의 특성에 맞는 최적의 방식을 선택할 수 있게 한다.

## 아키텍처

### Adapter 레이어 구조

```
Shell App
├── AppLoader (통합 진입점)
│   ├── loadApp(manifest) → RemoteApp
│   └── manifest.loadStrategy에 따라 Adapter 선택
│
├── Adapters/
│   ├── ModuleFederationAdapter  (loadStrategy: 'module-federation')
│   ├── SingleSpaAdapter         (loadStrategy: 'single-spa')
│   ├── IframeAdapter            (loadStrategy: 'iframe')
│   └── [확장 가능]              (loadStrategy: 'custom-xxx')
│
└── Remote Apps
    ├── App A (React, module-federation)
    ├── App B (Vue, single-spa)
    └── App C (외부 벤더, iframe)
```

### 핵심 인터페이스

**AppAdapter 인터페이스:** 모든 Adapter가 구현해야 하는 기본 인터페이스다. type 속성은 읽기 전용으로 해당 Adapter의 유형(예: 'module-federation', 'single-spa', 'iframe')을 나타낸다. load 메서드는 AppManifest 정보를 받아 RemoteApp 객체를 비동기로 반환한다. 선택적으로 preload 메서드로 리소스를 미리 로딩하거나, dispose 메서드로 리소스를 정리할 수 있다.

**AppManifest 인터페이스:** manifest에는 앱의 기본 정보(id, name, basePath, entry, routes, requiredPermissions, version, framework)와 함께 loadStrategy 필드가 포함된다. loadStrategy는 'module-federation', 'single-spa', 'iframe' 중 하나의 값을 가진다. iframe 방식의 경우 sandbox 속성과 allowedOrigin을 지정하는 iframeOptions를, Module Federation 방식의 경우 remoteName, exposedModule, sharedDeps를 지정하는 federationOptions를 추가로 포함할 수 있다.

### AppLoader 동작 방식

AppLoader 클래스는 각 통합 방식별 Adapter 인스턴스를 Map 자료구조로 관리한다. 생성 시 ModuleFederationAdapter, SingleSpaAdapter, IframeAdapter를 각각 해당 loadStrategy 키로 등록한다. loadApp 메서드가 호출되면 manifest의 loadStrategy 값을 기준으로 적합한 Adapter를 선택하고, 해당 Adapter의 load 메서드를 호출하여 RemoteApp을 반환한다. 지원하지 않는 loadStrategy인 경우 에러를 발생시킨다. registerAdapter 메서드를 통해 새로운 Adapter를 동적으로 등록하는 플러그인 방식의 확장도 가능하다.

### ModuleFederationAdapter 동작 방식

Module Federation Adapter의 load 메서드는 manifest에서 remoteName과 exposedModule 정보를 추출한 뒤, remoteEntry.js를 동적으로 로딩한다. 이후 Webpack의 공유 스코프를 초기화하고, window에 등록된 컨테이너에서 노출된 모듈을 가져와 RemoteApp 객체를 반환한다. preload 메서드는 link 요소를 생성하여 rel="preload" 속성으로 remoteEntry.js를 사전 로딩할 수 있다.

### IframeAdapter 동작 방식

iframe Adapter의 load 메서드는 IframeBridge 인스턴스를 생성하고, RemoteApp 계약을 구현하는 객체를 반환한다. bootstrap은 별도 동작 없이 빈 구현이며, mount 시 iframe 요소를 동적으로 생성하여 src, sandbox, 스타일을 설정한 뒤 지정된 컨테이너에 추가한다. iframe이 로드되면 IframeBridge를 통해 AppContext를 postMessage로 전달한다. unmount 시에는 bridge를 정리한다.

## 방식 선택 가이드라인

manifest의 `loadStrategy`를 어떻게 결정할지에 대한 가이드라인:

| 조건 | 권장 방식 | 이유 |
|------|----------|------|
| 내부 팀, React + Webpack 5 | `module-federation` | 디펜던시 공유로 최적 성능 |
| 내부 팀, Vue/Angular 등 이기종 | `single-spa` | 프레임워크 독립성 |
| 외부 벤더, 코드 리뷰 불가 | `iframe` | 보안 격리 필수 |
| 레거시 앱, 수정 불가 | `iframe` | 기존 코드 변경 없이 통합 |
| 마이그레이션 중 (과도기) | 현재 상태에 맞게 선택 | 향후 전환 가능 |

## 거버넌스 모델

Hybrid 방식은 기술적 유연성을 제공하지만, 조직 차원의 거버넌스가 없으면 혼란을 초래한다:

1. **기본 방식 지정**: 신규 앱의 기본 통합 방식을 하나 정한다 (예: Module Federation)
2. **예외 승인 프로세스**: 기본 방식을 따르지 않는 경우, 아키텍처 리뷰를 통해 승인
3. **마이그레이션 로드맵**: iframe으로 시작한 앱을 Module Federation으로 전환하는 일정 관리
4. **Adapter 유지보수 책임**: 각 Adapter의 오너십을 플랫폼 팀에 부여

## 성능 특성

| 항목 | 수치/특징 |
|------|----------|
| 초기 로딩 | Adapter 레이어 오버헤드 최소 (수 KB), 실제 성능은 선택된 방식에 종속 |
| 앱 전환 | 각 앱의 loadStrategy에 따라 상이 |
| 메모리 | Adapter 레지스트리 자체는 경미, 앱별 메모리는 방식에 종속 |
| 유지보수 | 3개 Adapter 코드 + 테스트 유지 필요 |

## 제약 사항 및 고려사항

1. **복잡도 비용**: Adapter 추상화 레이어를 구현·테스트·유지해야 하므로, 단일 방식 대비 초기 투자가 큼
2. **일관성 부재**: 같은 포털 내에서 앱마다 다른 성능/격리 특성을 보이므로, 사용자 경험이 일관되지 않을 수 있음
3. **디버깅 복잡도**: 문제 발생 시 Adapter 레이어, 특정 통합 방식, Remote App 중 어디가 원인인지 추적이 복잡
4. **테스트 매트릭스**: 모든 Adapter × 모든 브라우저 × 모든 시나리오의 조합 테스트 필요
