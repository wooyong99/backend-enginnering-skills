---
name: spec-to-tasks
description: Converts finalized requirements into structured development tasks. Use when requirements analysis is complete and policies/flows are clearly defined. Input is confirmed requirements with clear domain concepts, output is categorized tasks (API tasks, Domain tasks, Repository/Policy/Event tasks). Triggers when user says "태스크 생성", "개발 태스크로 변환", "구현 태스크 정리", or after requirements-analyst produces confirmed output.
user-invocable: true
---

# Spec to Tasks

확정된 요구사항을 Clean Architecture 레이어별 개발 태스크로 변환한다.

## Prerequisites

이 스킬 사용 전 다음이 준비되어야 한다:
- 요구사항 분석 완료 (requirements-analyst 또는 동등한 분석)
- 도메인 개념 정의 확정
- 비즈니스 정책/규칙 명확화
- 주요 플로우 확정

## Workflow

```
1. 입력 요구사항 검증
   ↓
2. 도메인 모델 태스크 도출
   ↓
3. Application 레이어 태스크 도출
   ↓
4. Infrastructure 레이어 태스크 도출
   ↓
5. API 레이어 태스크 도출
   ↓
6. 태스크 의존성 정렬
   ↓
7. 출력 생성
```

## Task Derivation Rules

### 1. Domain Layer Tasks

도메인 개념에서 태스크 도출:

| 도메인 개념 | 태스크 유형 | 예시 |
|-------------|-------------|------|
| 식별자 있는 개념 | Entity 생성 | `User`, `Order`, `Payment` |
| 불변 속성 그룹 | Value Object 생성 | `Money`, `Address`, `DateRange` |
| 도메인 불변식 | 검증 로직 구현 | `주문금액 > 0`, `재고 >= 주문수량` |
| 상태 전이 규칙 | 상태 머신 구현 | `PENDING → CONFIRMED → SHIPPED` |
| 복수 엔티티 조합 로직 | Domain Service 생성 | `PriceCalculator`, `InventoryChecker` |
| 비즈니스 이벤트 | Domain Event 정의 | `OrderPlaced`, `PaymentCompleted` |

### 2. Application Layer Tasks

비즈니스 플로우에서 태스크 도출:

| 플로우 특성 | 태스크 유형 | 예시 |
|-------------|-------------|------|
| 단일 비즈니스 연산 | Use Case 생성 | `CreateOrderUseCase` |
| 입력 데이터 구조 | Command 정의 | `CreateOrderCommand` |
| 출력 데이터 구조 | Result 정의 | `CreateOrderResult` |
| 외부 시스템 의존 | Port 인터페이스 정의 | `PaymentGatewayPort` |
| 도메인 이벤트 처리 | Event Handler 생성 | `OrderPlacedHandler` |

### 3. Infrastructure Layer Tasks

외부 의존성에서 태스크 도출:

| 외부 의존성 | 태스크 유형 | 예시 |
|-------------|-------------|------|
| 데이터 영속화 | Repository 구현 | `JpaOrderRepository` |
| 외부 API 연동 | Adapter 구현 | `TossPaymentAdapter` |
| 메시징 시스템 | Publisher/Consumer 구현 | `KafkaOrderEventPublisher` |
| 캐싱 | Cache Adapter 구현 | `RedisProductCacheAdapter` |

### 4. API Layer Tasks

사용자 인터페이스에서 태스크 도출:

| 인터페이스 요소 | 태스크 유형 | 예시 |
|-----------------|-------------|------|
| API 엔드포인트 | Controller 생성 | `OrderController` |
| 요청 데이터 | Request DTO 정의 | `CreateOrderRequest` |
| 응답 데이터 | Response DTO 정의 | `OrderResponse` |
| 에러 응답 | Exception Handler 정의 | `OrderExceptionHandler` |

## Output Format

태스크는 다음 형식으로 출력한다:

```markdown
## 개발 태스크 목록

### 구현 순서
[권장 구현 순서와 의존성 설명]

---

### Domain Layer Tasks

#### Entities
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| D-E-001 | `Order` Entity 생성 | 주문 도메인 객체 | - |

#### Value Objects
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| D-V-001 | `Money` VO 생성 | 금액 값 객체 | - |

#### Domain Services
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| D-S-001 | `PriceCalculator` 생성 | 가격 계산 서비스 | D-E-001, D-V-001 |

#### Domain Events
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| D-EV-001 | `OrderPlaced` 이벤트 정의 | 주문 생성 이벤트 | D-E-001 |

---

### Application Layer Tasks

#### Use Cases
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| A-UC-001 | `CreateOrderUseCase` 생성 | 주문 생성 유스케이스 | D-E-001, A-P-001 |

#### Commands & Results
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| A-C-001 | `CreateOrderCommand` 정의 | 주문 생성 입력 | - |

#### Ports
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| A-P-001 | `OrderRepository` Port 정의 | 주문 저장소 인터페이스 | D-E-001 |

#### Event Handlers
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| A-EH-001 | `OrderPlacedHandler` 생성 | 주문 생성 이벤트 처리 | D-EV-001 |

---

### Infrastructure Layer Tasks

#### Repository Implementations
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| I-R-001 | `JpaOrderRepository` 구현 | JPA 기반 주문 저장소 | A-P-001 |

#### External Adapters
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| I-A-001 | `PaymentAdapter` 구현 | 결제 시스템 연동 | A-P-002 |

#### Event Publishers
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| I-EP-001 | `OrderEventPublisher` 구현 | 주문 이벤트 발행 | D-EV-001 |

---

### API Layer Tasks

#### Controllers
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| P-C-001 | `OrderController` 생성 | 주문 API 컨트롤러 | A-UC-001 |

#### DTOs
| ID | 태스크 | 설명 | 선행 태스크 |
|----|--------|------|-------------|
| P-D-001 | `CreateOrderRequest` 정의 | 주문 생성 요청 DTO | - |

---

### Summary

| 레이어 | 태스크 수 |
|--------|-----------|
| Domain | N |
| Application | N |
| Infrastructure | N |
| API | N |
| **Total** | **N** |
```

## Task ID Convention

- `D-E-XXX`: Domain Entity
- `D-V-XXX`: Domain Value Object
- `D-S-XXX`: Domain Service
- `D-EV-XXX`: Domain Event
- `A-UC-XXX`: Application Use Case
- `A-C-XXX`: Application Command/Result
- `A-P-XXX`: Application Port
- `A-EH-XXX`: Application Event Handler
- `I-R-XXX`: Infrastructure Repository
- `I-A-XXX`: Infrastructure Adapter
- `I-EP-XXX`: Infrastructure Event Publisher
- `P-C-XXX`: Presentation Controller
- `P-D-XXX`: Presentation DTO

## Dependency Ordering

태스크 의존성 정렬 원칙:

1. **Domain First**: 도메인 레이어 태스크를 먼저 배치
2. **Inside-Out**: 내부 레이어에서 외부 레이어 순으로
3. **Port Before Adapter**: Port 정의 후 구현체 태스크
4. **Event Before Handler**: 이벤트 정의 후 핸들러 태스크

```
Domain (Entity, VO)
  → Domain (Service, Event)
    → Application (Port, Command)
      → Application (UseCase, Handler)
        → Infrastructure (Repository, Adapter)
          → API (Controller, DTO)
```

## Quality Checklist

태스크 생성 후 검증:

- [ ] 모든 도메인 개념이 태스크로 변환되었는가?
- [ ] 모든 비즈니스 플로우가 Use Case로 표현되었는가?
- [ ] 외부 의존성마다 Port가 정의되었는가?
- [ ] 태스크 간 의존성이 명확한가?
- [ ] 순환 의존성이 없는가?
- [ ] 구현 순서가 의존성을 존중하는가?

## Reference

상세 태스크 분류 기준: [references/task-categories.md](references/task-categories.md)
