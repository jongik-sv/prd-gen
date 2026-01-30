---
category: design-system
label: Design System 전략
summary: Micro Frontend 환경에서 시각적 일관성을 유지하기 위한 Design System 제공 범위와 방식을 결정한다.
context:
  - 여러 팀이 독립적으로 개발하는 Remote App들이 하나의 포털에서 사용자에게 일관된 UI/UX를 제공해야 한다
  - Design System은 색상, 타이포그래피, 간격 등의 Design Token과 버튼, 테이블 등의 UI 컴포넌트로 구성된다
  - 프레임워크가 다양할수록 Design System 배포 전략이 중요해진다
dependencies:
  - 01-mf-integration
---

## 배경

Micro Frontend 환경에서 각 Remote App이 제각각의 UI를 구현하면, 사용자는 같은 포털 안에서 서로 다른 버튼 스타일, 다른 폰트, 다른 색상을 경험하게 된다. 이는 브랜드 일관성을 해치고 학습 비용을 높인다.

Design System은 이 문제를 해결하기 위한 공유 자산으로, 크게 두 계층으로 나뉜다:

### 2-tier 구조

| 계층 | 내용 | 프레임워크 의존성 | 배포 형태 |
|------|------|-----------------|----------|
| **Tier 1: Design Token** | 색상, 타이포그래피, 간격, 그림자 등의 원자적 값 | 없음 (순수 CSS 변수) | CSS 파일 또는 JSON |
| **Tier 2: UI 컴포넌트** | 버튼, 입력 필드, 테이블, 모달 등의 완성된 컴포넌트 | 프레임워크별 구현 | npm 패키지 |

Tier 1은 모든 프레임워크에서 사용 가능한 보편적 기반이고, Tier 2는 각 프레임워크의 네이티브 경험을 제공하는 실질적인 UI 라이브러리다.

### Design Token 예시

Design Token은 CSS 변수로 정의되어 모든 프레임워크에서 참조할 수 있다:

- `--color-primary`: 브랜드 기본 색상
- `--font-family`: 기본 폰트 패밀리
- `--space-md`: 기본 간격 단위
- `--radius-md`: 기본 모서리 둥글기
- `--shadow-md`: 기본 그림자

이 토큰들은 테마 전환(다크 모드 등)의 기반이 되며, 토큰 값만 변경하면 전체 포털의 테마를 일괄 전환할 수 있다.

### 선택지 요약

| 선택지 | 핵심 전략 | 적용 대상 |
|--------|----------|----------|
| React 네이티브 컴포넌트 | Tier 1 + Tier 2를 React 전용으로 구축 | 전 앱이 React |
| Vue 네이티브 컴포넌트 | Tier 1 + Tier 2를 Vue 전용으로 구축 | 전 앱이 Vue |
| Web Components | Tier 1 + Tier 2를 Web Components로 구축 | 이기종 프레임워크 혼재 |
| Design Token만 | Tier 1만 제공, Tier 2는 각 팀이 자체 구현 | 팀 자율성 우선 |
