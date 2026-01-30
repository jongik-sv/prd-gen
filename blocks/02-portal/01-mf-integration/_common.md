---
category: mf-integration
label: Micro Frontend 통합 방식
summary: Shell App과 Remote App을 연결하는 Micro Frontend 통합 전략을 결정한다.
context:
  - 포털 Shell App은 여러 개의 독립 배포 가능한 Remote App을 하나의 화면에 통합한다
  - 통합 방식에 따라 성능, 격리 수준, 프레임워크 호환성, 개발 경험이 크게 달라진다
  - Shell App은 Contract 기반 설계를 채택하여, 통합 방식에 무관한 계약(bootstrap/mount/unmount)을 정의한다
dependencies:
  - 02-security-isolation
  - 03-design-system
---

## 배경

Micro Frontend(MF) 아키텍처는 대규모 프론트엔드 애플리케이션을 독립적으로 개발·배포 가능한 단위로 분할하는 접근이다. 포털 시스템에서는 Shell App이 글로벌 레이아웃과 인증을 담당하고, 각 업무 영역은 Remote App으로 분리하여 독립 팀이 개발한다.

### 핵심 문제

통합 방식을 결정할 때 고려해야 할 핵심 트레이드오프는 다음과 같다:

| 축 | 설명 |
|---|---|
| **공유 vs 격리** | 디펜던시(React 등)를 공유하면 번들 크기가 줄지만, 버전 충돌 위험이 커진다 |
| **성능 vs 안전** | 같은 JS 컨텍스트에서 실행하면 빠르지만, 한 앱의 오류가 전체에 영향을 줄 수 있다 |
| **유연성 vs 복잡성** | 여러 프레임워크를 지원하면 자유도가 높지만, 빌드/배포 파이프라인이 복잡해진다 |

### Shell App Contract

어떤 통합 방식을 선택하든, Remote App은 다음 계약을 준수해야 한다. Remote App 인터페이스는 세 가지 생명주기 메서드를 포함한다. bootstrap 메서드는 앱 초기화를 수행하고, mount 메서드는 지정된 HTML 컨테이너 요소와 AppContext를 전달받아 앱을 화면에 렌더링하며, unmount 메서드는 앱을 정리하고 해제한다. 모든 메서드는 비동기(Promise)로 동작한다. 또한 manifest 객체를 통해 앱의 고유 식별자(id), 이름(name), 라우트 설정(routes), 필요 권한 목록(permissions), 버전(version) 정보를 선언해야 한다.

Shell App은 Adapter 레이어를 통해 각 통합 방식의 차이를 흡수하며, Remote App 입장에서는 동일한 계약만 구현하면 된다.

### 선택지 요약

| 선택지 | 핵심 특징 | 프레임워크 호환 | 디펜던시 공유 |
|--------|----------|---------------|-------------|
| Module Federation | Webpack 5 빌드 타임 통합 | 동일 프레임워크 최적 | O |
| Single-SPA | 런타임 프레임워크 독립 통합 | 완전 독립 | X |
| iframe | 완전 격리 통합 | 완전 독립 | X |
| Hybrid (Adapter) | 모든 방식 수용 | 완전 독립 | 방식별 상이 |
