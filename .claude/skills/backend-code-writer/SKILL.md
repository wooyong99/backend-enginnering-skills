---
name: backend-code-writer
description: Clean Architecture와 DDD 기반 백엔드 코드 작성 가이드. 백엔드 코드 작성, 새 기능 구현, Use Case 설계, 도메인 모델링, 테스트 코드 작성, 인프라 레이어 구현 시 사용. Java/Kotlin 백엔드 프로젝트에서 코드 품질과 아키텍처 일관성을 유지하기 위한 가이드라인 제공.
context: fork
agent: backend-architect
user-invocable: true
---

# Backend Code Writer

Clean Architecture + Domain-Driven Design 기반 백엔드 코드 작성 가이드.

## Reference Documents

작업 유형에 따라 필요한 참조 문서를 읽어 가이드라인을 준수한다.

### 핵심 원칙 (모든 작업 시 참조)

| 파일                                                                                  | 내용                                    | 언제 읽는가                 |
| ------------------------------------------------------------------------------------- | --------------------------------------- | --------------------------- |
| [BACKEND_DEVELOPMENT_PRINCIPLES.md](references/BACKEND_DEVELOPMENT_PRINCIPLES.md)     | 레이어 분리, 의존성 역전, 도메인 순수성 | 코드 작성 전 기본 원칙 확인 |
| [CHANGE_DRIVEN_ARCHITECTURE_GUIDE.md](references/CHANGE_DRIVEN_ARCHITECTURE_GUIDE.md) | 변경 이유 기반 설계 판단                | 코드 배치 위치 결정 시      |

### 도메인 레이어 작업

| 파일                                                                        | 내용                                      | 언제 읽는가              |
| --------------------------------------------------------------------------- | ----------------------------------------- | ------------------------ |
| [DOMAIN_DESIGN_GUIDE.md](references/DOMAIN_DESIGN_GUIDE.md)                 | 엔티티, 값 객체, Aggregate, 도메인 서비스 | 도메인 모델 설계/구현 시 |
| [COMPONENT_DESIGN_PRINCIPLES.md](references/COMPONENT_DESIGN_PRINCIPLES.md) | 컴포넌트 응집도/결합 원칙                 | 패키지 구조 결정 시      |

### Application 레이어 작업

| 파일                                              | 내용                                | 언제 읽는가             |
| ------------------------------------------------- | ----------------------------------- | ----------------------- |
| [USE_CASE_GUIDE.md](references/USE_CASE_GUIDE.md) | Use Case 구조, Command/Result, Port | Use Case 클래스 작성 시 |

### Infrastructure 레이어 작업

| 파일                                                                                          | 내용                         | 언제 읽는가           |
| --------------------------------------------------------------------------------------------- | ---------------------------- | --------------------- |
| [INFRASTRUCTURE_API_INTEGRATION_GUIDE.md](references/INFRASTRUCTURE_API_INTEGRATION_GUIDE.md) | Adapter, ApiClient, DTO 구조 | 외부 API 연동 구현 시 |

### 테스트 작성

| 파일                                                | 내용                                     | 언제 읽는가         |
| --------------------------------------------------- | ---------------------------------------- | ------------------- |
| [TEST_CODE_GUIDE.md](references/TEST_CODE_GUIDE.md) | FIRST 원칙, 테스트 피라미드, 테스트 더블 | 테스트 코드 작성 시 |

## Workflow

```
1. 작업 유형 파악
   ↓
2. 해당하는 참조 문서 읽기
   ↓
3. 가이드라인에 따라 코드 작성
   ↓
4. 테스트 코드 작성 (TEST_CODE_GUIDE.md 참조)
```

## Quick Reference: 레이어별 책임

```
Presentation Layer (API)
├── Controller: 요청/응답 변환, Use Case 호출
└── DTO: 외부 인터페이스 데이터 구조

Application Layer (Use Case)
├── UseCase: 흐름 제어, 트랜잭션 관리
├── Command/Result: 입출력 데이터
└── Port: 외부 의존성 인터페이스

Domain Layer
├── Entity: 식별자 있는 도메인 객체
├── Value Object: 불변 값 객체
├── Domain Service: 여러 엔티티 조합 로직
└── Domain Event: 도메인 이벤트

Infrastructure Layer
├── Adapter: Port 구현체
├── ApiClient: 외부 API 호출
└── Repository Impl: 영속성 구현
```

## Quick Reference: 코드 배치 판단

```
"이 코드는 왜 변경되는가?"

비즈니스 규칙 변경 → Domain Layer
비즈니스 플로우 변경 → Application Layer
외부 시스템 변경 → Infrastructure Layer
UI/프로토콜 변경 → Presentation Layer
```
