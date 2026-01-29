# 6인 전문가 브레인스토밍 - 토의 주제 목차

## 프롬프트
  - 앞으로 여러 프로젝트를 개발할 것인데 각 프로젝트들은 각기 다른 모습을 가질        
  것이고 다른 기술 스택을 사용할 거야. 그래서 각각을 조합해서 PRD, TRD를 기준으로   
  포털, 권한관리, 마스터관리에 대한 설계까지 하는 것이 목표야. 각각 어떤 부분까지   
  검토해야 하는지 SWE의 각 분야 전문가(5명)로 빙의해서 아이디어롤 도출해줘.
  - AI전문가(앤드류 응, 카파시), DB전문가(유명인), 아키텍쳐 전문가, SW 전문가,        
  프로젝트 전문가 등 5명이 아니고 6명이 진행해줘.                                   
  아이디어 도출을 위한 토의를 해줘.                                                 
  브레인스토밍을 시작해. 

## 목표
- Claude Code, Gemini 등 AI 에이전트가 바로 시스템을 개발할 수 있는 수준의 PRD/TRD 산출
- 포털, 권한관리, 마스터관리 3개 영역의 설계 완성

## 참석 전문가
| # | 분야 | 인물 | 역할 |
|---|------|------|------|
| 1 | AI | Andrew Ng + Andrej Karpathy | AI/ML 관점 제안 |
| 2 | DB | Martin Kleppmann | 데이터 모델/저장소 설계 |
| 3 | 아키텍처 | Martin Fowler | 시스템 구조/패턴 설계 |
| 4 | SW | Robert C. Martin (Uncle Bob) | 구현 품질/Contract 정의 |
| 5 | PM | Kent Beck | 사회자, 일정/리스크/프로세스 |
| 6 | UX/보안 | Jakob Nielsen + Bruce Schneier | 사용성/보안 검증 |
| 7 | 문서화 | **Donald Knuth** 스타일 | 토의 결과를 구조적으로 정리하여 AI-executable 수준의 PRD/TRD 문서로 산출 |

---

## Part 1: 전체 아키텍처 토의 (완료)
- [x] 1.1 전문가별 핵심 관점 오프닝
- [x] 1.2 교차 토론 - 권한 모델 (RBAC vs ABAC)
- [x] 1.3 교차 토론 - 마스터 데이터 통합 전략
- [x] 1.4 교차 토론 - 다중 기술스택 포털 통합 방식
- [x] 1.5 누락 관점 보완 (DevOps, DX, 거버넌스, 성능, 마이그레이션, 확장)

## Part 2: 포털 설계 심화 토의 (완료)
- [x] 2.1 Micro Frontend 통합 방식 결정
- [x] 2.2 Shell App 책임 범위 및 Contract 정의
- [x] 2.3 App Registry & 라우팅 설계
- [x] 2.4 UX 최적화 (Skeleton, 프리페칭, 상태 캐싱)
- [x] 2.5 Design System 전략 (Design Token + 컴포넌트)
- [x] 2.6 AI-Executable 문서 수준 정의

## Part 3: 권한관리 설계 심화 토의 (완료)
- [x] 3.1 권한 모델 상세 설계 (RBAC + Project Scope 구조)
- [x] 3.2 인증 흐름 설계 (SSO, OAuth2/OIDC, Token 생명주기)
- [x] 3.3 권한 DB 스키마 설계 (ERD)
- [x] 3.4 권한 체크 API 설계 (엔드포인트, 입출력)
- [x] 3.5 Policy Engine 연동 설계 (OPA)
- [x] 3.6 권한 관리 UI/UX 설계 (역할 매트릭스, 위임, 승인)
- [x] 3.7 감사 로그 설계 (Audit Trail)

## Part 4: 마스터관리 설계 심화 토의 (완료)
- [x] 4.1 마스터 데이터 모델 설계 (코드 체계, 계층, 생명주기)
- [x] 4.2 MDM Hub 서비스 API 설계
- [x] 4.3 마스터 DB 스키마 설계 (ERD)
- [x] 4.4 데이터 동기화 전략 (Push/Pull/Hybrid)
- [x] 4.5 캐싱 전략 (2-tier, 무효화)
- [x] 4.6 변경 거버넌스 워크플로우 (요청-검토-승인)
- [x] 4.7 벌크 처리 설계 (Excel Import/Export)
- [x] 4.8 데이터 품질 관리 (중복 탐지, AI 적용)

## Part 5: 통합 / 공통 설계 토의 (완료)
- [x] 5.1 전체 시스템 아키텍처 다이어그램
- [x] 5.2 API Gateway / BFF 설계
- [x] 5.3 공통 SDK 명세 (Auth SDK, Master SDK, Portal SDK)
- [x] 5.4 CI/CD 파이프라인 설계
- [x] 5.5 관찰가능성 설계 (Logging, Metrics, Tracing)
- [x] 5.6 테스트 전략 (Contract, E2E, 권한 시나리오)
- [x] 5.7 개발자 온보딩 / DX 설계

## Part 6: PRD 문서 작성 (완료)
- [x] 6.1 비전 & 목표
- [x] 6.2 사용자 페르소나
- [x] 6.3 기능 요구사항 (포털 / 권한 / 마스터)
- [x] 6.4 비기능 요구사항 (성능, 가용성, 보안)
- [x] 6.5 MoSCoW 우선순위
- [x] 6.6 릴리즈 로드맵

## Part 7: TRD 문서 작성 (완료)
- [x] 7.1 시스템 아키텍처 (다이어그램 + 설명)
- [x] 7.2 디렉토리 구조 명세
- [x] 7.3 TypeScript 인터페이스/타입 정의
- [x] 7.4 API 엔드포인트 + 입출력 스키마 (OpenAPI)
- [x] 7.5 DB 스키마 (SQL DDL)
- [x] 7.6 시퀀스 다이어그램 (주요 흐름)
- [x] 7.7 환경 설정 (의존성, 환경변수)
- [x] 7.8 테스트 시나리오

---

## 진행 상태
- **현재**: Part 1~7 전체 완료
- **상태**: 모든 브레인스토밍 토의 및 PRD/TRD 문서 작성 완료

## 산출물 파일 목록
| Part | 파일명 | 상태 |
|------|--------|------|
| Part 1 | `brainstorming-part1-architecture.md` | 완료 |
| Part 2 | `brainstorming-part2-portal-design.md` | 완료 |
| Part 3 | `brainstorming-part3-auth.md` | 완료 |
| Part 4 | `brainstorming-part4-master.md` | 완료 |
| Part 5 | `brainstorming-part5-integration.md` | 완료 |
| Part 6 | `brainstorming-part6-prd.md` | 완료 |
| Part 7 | `brainstorming-part7-trd.md` | 완료 |
