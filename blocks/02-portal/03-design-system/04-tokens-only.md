---
id: ds-tokens-only
category: design-system
label: Design Token만 제공
summary: CSS 변수 기반 Design Token(Tier 1)만 중앙에서 제공하고, UI 컴포넌트(Tier 2)는 각 팀이 자율적으로 구현한다.
pros:
  - 각 팀이 선호하는 UI 프레임워크(Material UI, Ant Design, Element Plus 등)를 자유롭게 사용 가능
  - 중앙 Design System 팀의 부담 최소화 (토큰 관리만 집중)
  - 토큰만 맞추면 색상, 타이포그래피, 간격의 일관성은 유지
  - 초기 구축 비용이 가장 낮음
cons:
  - 컴포넌트 수준의 UI 일관성을 보장할 수 없음 (버튼 모양, 테이블 동작 등이 앱마다 다를 수 있음)
  - 각 팀이 동일한 컴포넌트를 중복 구현하여 전체적인 개발 비용 증가
  - 접근성(a11y) 품질이 팀마다 달라질 수 있음
  - UX 일관성 검증이 어렵고, 시간이 지남에 따라 UI가 파편화될 가능성
bestFor:
  - 각 팀의 기술 자율성이 최우선인 조직
  - 이미 각 팀이 자체 UI 라이브러리를 보유하고 있는 경우
  - 초기에 빠르게 포털을 구축하고, 추후 Design System을 점진적으로 확장하려는 경우
---

## 개요

Design Token만 제공하는 전략은 Tier 1(토큰)만 중앙에서 관리하고, Tier 2(컴포넌트)는 의도적으로 제공하지 않는 방식이다. 각 Remote App 팀은 `@portal/design-tokens` 패키지의 CSS 변수를 참조하여 색상, 폰트, 간격을 맞추되, 버튼, 테이블, 모달 등의 컴포넌트는 자체적으로 구현하거나 서드파티 라이브러리를 사용한다.

이 접근의 핵심 철학은 **최소한의 제약으로 최대한의 일관성**을 추구하는 것이다. 토큰만 공유해도 시각적으로 "같은 포털에 속한 앱"이라는 느낌을 줄 수 있으며, 팀의 기술적 자유를 최대한 보장한다.

## Design Token 패키지 구성

### 배포 형태

| 형태 | 대상 | 내용 |
|------|------|------|
| **CSS 파일** | 모든 앱 | `:root`에 CSS 변수로 정의된 전체 토큰 |
| **JSON 파일** | 빌드 도구, 스크립트 | 토큰의 원본 데이터 (이름, 값, 설명) |
| **SCSS 변수** | SCSS 사용 앱 | `$color-primary: var(--color-primary)` 형태 |
| **JS 상수** | CSS-in-JS 사용 앱 | `export const colorPrimary = 'var(--color-primary)'` 형태 |

### 토큰 카테고리

| 카테고리 | 토큰 예시 | 용도 |
|---------|----------|------|
| **Color** | primary, secondary, error, success, warning, text, background | 색상 체계 |
| **Typography** | font-family, font-size (xs~xl), font-weight, line-height | 글꼴 체계 |
| **Spacing** | space (xs~xl) | 요소 간 간격 |
| **Border** | radius (sm~lg), border-width, border-color | 테두리 스타일 |
| **Shadow** | shadow (sm~lg) | 그림자 깊이 |
| **Z-index** | z-index (base, dropdown, modal, toast) | 겹침 순서 |
| **Animation** | duration (fast, normal, slow), easing | 트랜지션 설정 |

## 테마 전환

CSS 변수 기반이므로 테마 전환이 단순하다. `data-theme` 속성을 HTML 루트 요소에 적용하면, CSS에서 해당 속성에 따라 변수 값을 오버라이드한다. Light/Dark 모드 전환 시 토큰 값만 교체되므로, 각 팀의 컴포넌트가 자동으로 테마에 맞춰진다.

## 일관성 확보 방법

컴포넌트 없이 토큰만으로 일관성을 유지하기 위한 보완 장치:

| 장치 | 내용 | 효과 |
|------|------|------|
| **UX 가이드라인 문서** | 버튼 크기, 간격 규칙, 타이포그래피 사용법 등을 문서로 제공 | 디자인 방향성 통일 |
| **Figma 디자인 키트** | 토큰에 맞춘 Figma 컴포넌트 제공 (구현은 팀 자율) | 디자이너 수준의 일관성 |
| **UX 리뷰 프로세스** | 정기적으로 각 앱의 UI를 리뷰하여 가이드라인 준수 확인 | 시각적 품질 관리 |
| **시각적 회귀 테스트** | 스크린샷 비교로 의도치 않은 UI 변경 감지 | 자동화된 품질 관리 |

## 서드파티 UI 라이브러리 활용

각 팀이 서드파티 라이브러리를 사용할 때, 토큰과 연결하는 전략:

| 라이브러리 | 토큰 연결 방법 |
|-----------|---------------|
| **Material UI (React)** | createTheme에서 CSS 변수를 참조하는 커스텀 테마 생성 |
| **Ant Design (React)** | ConfigProvider의 theme 설정에서 토큰 값 매핑 |
| **Element Plus (Vue)** | SCSS 변수 오버라이드 또는 CSS 변수 직접 사용 |
| **Vuetify (Vue)** | createVuetify의 theme 설정에서 토큰 값 매핑 |

이렇게 하면 각 팀이 다른 UI 라이브러리를 사용하더라도, 기본 색상/폰트/간격은 동일하게 유지된다.

## 점진적 확장 경로

Token-only 전략은 추후 컴포넌트 라이브러리를 도입하는 것을 배제하지 않는다:

```
Phase 1: Design Token만 배포, 각 팀 자율 구현
    ↓
Phase 2: 가장 많이 중복되는 컴포넌트(버튼, 입력, 테이블)를 공통 라이브러리로 추출
    ↓
Phase 3: 전체 컴포넌트 라이브러리로 확장 (React-UI 또는 Web Components)
```

이 접근은 초기 비용을 최소화하면서, 실제 필요에 따라 점진적으로 Design System을 성장시킬 수 있다.

## 제약 사항 및 고려사항

1. **UI 파편화 위험**: 시간이 지남에 따라 앱마다 버튼 모양, 테이블 동작, 모달 스타일 등이 달라질 수 있음
2. **중복 구현**: 동일한 DataGrid, DatePicker 등을 여러 팀이 각각 구현하여 전체 개발 비용 증가
3. **접근성 편차**: 접근성 구현 수준이 팀 역량에 의존하여 앱마다 달라질 수 있음
4. **온보딩 비용**: 새 팀이 합류할 때 "토큰은 있지만 컴포넌트는 없다"면, 무엇을 어떻게 구현해야 하는지 가이드가 부족할 수 있음
