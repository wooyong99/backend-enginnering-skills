# Task Categories Reference

태스크 분류 상세 기준과 예시.

## Domain Layer Task Details

### Entity Tasks

**판단 기준**
- 고유 식별자(ID)가 필요한 개념
- 생명주기(생성, 수정, 삭제)가 있는 개념
- 상태 변화를 추적해야 하는 개념

**태스크 상세 항목**
```
Entity Task 상세:
├── 식별자 설계 (UUID, Sequential, Natural Key)
├── 필수 속성 정의
├── 불변식(Invariant) 구현
├── 상태 전이 메서드
├── 도메인 이벤트 발행 지점
└── equals/hashCode (식별자 기반)
```

**예시**
| 도메인 개념 | Entity 후보 | 주요 불변식 |
|-------------|-------------|-------------|
| 주문 | `Order` | 총액 > 0, 상품 1개 이상 |
| 회원 | `Member` | 이메일 유일, 비밀번호 규칙 |
| 상품 | `Product` | 가격 > 0, 재고 >= 0 |
| 결제 | `Payment` | 금액 = 주문금액 |

### Value Object Tasks

**판단 기준**
- 식별자 없이 속성값으로만 동등성 판단
- 불변(Immutable)
- 개념적으로 하나의 단위

**태스크 상세 항목**
```
Value Object Task 상세:
├── 속성 정의 (final)
├── 생성자 검증 로직
├── 정적 팩토리 메서드
├── 연산 메서드 (새 VO 반환)
└── equals/hashCode (모든 속성 기반)
```

**예시**
| 개념 | Value Object | 포함 속성 | 검증 규칙 |
|------|--------------|-----------|-----------|
| 금액 | `Money` | amount, currency | amount >= 0 |
| 주소 | `Address` | city, street, zipCode | zipCode 형식 |
| 기간 | `DateRange` | start, end | start <= end |
| 수량 | `Quantity` | value | value > 0 |

### Domain Service Tasks

**판단 기준**
- 여러 Entity/VO를 조합하는 로직
- 특정 Entity에 속하지 않는 비즈니스 규칙
- 상태를 갖지 않는 순수 연산

**태스크 상세 항목**
```
Domain Service Task 상세:
├── 입력 파라미터 정의
├── 비즈니스 로직 구현
├── 결과 반환 타입
└── 예외 상황 정의
```

**예시**
| 비즈니스 로직 | Domain Service | 관련 Entity |
|---------------|----------------|-------------|
| 할인 계산 | `DiscountCalculator` | Order, Coupon, Member |
| 배송비 계산 | `ShippingFeeCalculator` | Order, Address |
| 재고 확인 | `InventoryChecker` | Product, OrderItem |

### Domain Event Tasks

**판단 기준**
- 도메인에서 발생한 중요한 사건
- 다른 도메인/시스템이 관심 가질 사건
- 과거형 명명 (OrderPlaced, PaymentCompleted)

**태스크 상세 항목**
```
Domain Event Task 상세:
├── 이벤트 속성 정의
├── 발생 시점 (occurredAt)
├── 발행 주체 식별자
└── 직렬화 방식
```

**예시**
| 비즈니스 사건 | Domain Event | 포함 데이터 |
|---------------|--------------|-------------|
| 주문 생성 | `OrderPlaced` | orderId, memberId, totalAmount |
| 결제 완료 | `PaymentCompleted` | paymentId, orderId, amount |
| 배송 시작 | `ShipmentStarted` | orderId, trackingNumber |

---

## Application Layer Task Details

### Use Case Tasks

**판단 기준**
- 하나의 비즈니스 시나리오
- 사용자 관점의 단일 작업 단위
- 트랜잭션 경계

**태스크 상세 항목**
```
Use Case Task 상세:
├── Command 정의
├── Result 정의
├── Port 의존성 명시
├── 실행 흐름 구현
├── 트랜잭션 범위 결정
└── 도메인 이벤트 발행
```

**Use Case 명명 규칙**
```
[동사][대상]UseCase

Create  → 생성: CreateOrderUseCase
Update  → 수정: UpdateMemberUseCase
Delete  → 삭제: DeleteProductUseCase
Get     → 단건 조회: GetOrderUseCase
List    → 목록 조회: ListOrdersUseCase
Process → 처리: ProcessPaymentUseCase
Cancel  → 취소: CancelOrderUseCase
```

### Command & Result Tasks

**Command 태스크**
```
Command Task 상세:
├── 필수 필드 정의
├── 선택 필드 정의
├── 입력 검증 규칙
└── 정적 팩토리 메서드
```

**Result 태스크**
```
Result Task 상세:
├── 성공 시 반환 데이터
├── 실패 유형 정의
└── 정적 팩토리 메서드 (success, failure)
```

### Port Tasks

**판단 기준**
- Use Case가 필요로 하는 외부 기능
- 구현 세부사항 은닉
- 의존성 역전을 위한 추상화

**Port 유형별 태스크**
```
Repository Port:
├── save(entity): Entity
├── findById(id): Optional<Entity>
├── findAll(criteria): List<Entity>
└── delete(entity): void

Gateway Port (외부 API):
├── 요청 메서드 정의
├── 응답 타입 정의
└── 예외 상황 정의

EventPublisher Port:
├── publish(event): void
└── 발행 보장 수준 명시
```

### Event Handler Tasks

**태스크 상세 항목**
```
Event Handler Task 상세:
├── 처리할 이벤트 타입
├── 처리 로직 구현
├── 실패 시 재시도 정책
└── 멱등성 보장 방식
```

---

## Infrastructure Layer Task Details

### Repository Implementation Tasks

**태스크 상세 항목**
```
Repository Impl Task 상세:
├── JPA Entity 매핑
├── Query 메서드 구현
├── 도메인 Entity 변환
└── 영속성 컨텍스트 관리
```

### External Adapter Tasks

**태스크 상세 항목**
```
External Adapter Task 상세:
├── API Client 구현
├── 요청 DTO 매핑
├── 응답 DTO 매핑
├── 에러 핸들링
├── 타임아웃/재시도 설정
└── 서킷브레이커 적용
```

### Event Publisher Implementation Tasks

**태스크 상세 항목**
```
Event Publisher Impl Task 상세:
├── 메시지 브로커 연동 (Kafka, RabbitMQ 등)
├── 이벤트 직렬화
├── 발행 실패 처리
└── 아웃박스 패턴 적용 (필요시)
```

---

## API Layer Task Details

### Controller Tasks

**태스크 상세 항목**
```
Controller Task 상세:
├── 엔드포인트 정의 (HTTP Method, Path)
├── Request DTO 바인딩
├── Use Case 호출
├── Response DTO 변환
└── HTTP 상태 코드 매핑
```

**REST 엔드포인트 패턴**
```
POST   /orders          → CreateOrderController
GET    /orders/{id}     → GetOrderController
GET    /orders          → ListOrdersController
PUT    /orders/{id}     → UpdateOrderController
DELETE /orders/{id}     → DeleteOrderController
POST   /orders/{id}/cancel → CancelOrderController
```

### DTO Tasks

**Request DTO 태스크**
```
Request DTO Task 상세:
├── 필드 정의
├── 검증 어노테이션 (@NotNull, @Size 등)
├── Command 변환 메서드
└── API 문서 어노테이션
```

**Response DTO 태스크**
```
Response DTO Task 상세:
├── 필드 정의
├── Entity/Result 변환 메서드
├── 직렬화 설정
└── API 문서 어노테이션
```

---

## Cross-Cutting Task Patterns

### 트랜잭션 관련 태스크

| 상황 | 추가 태스크 |
|------|-------------|
| 단일 Aggregate 수정 | 기본 @Transactional |
| 복수 Aggregate 수정 | Saga/이벤트 기반 처리 태스크 |
| 외부 API 호출 포함 | 보상 트랜잭션 태스크 |

### 동시성 관련 태스크

| 상황 | 추가 태스크 |
|------|-------------|
| 재고 차감 | 낙관적/비관적 락 구현 태스크 |
| 동시 수정 가능 | Version 필드 추가 태스크 |
| 분산 환경 | 분산 락 구현 태스크 |

### 성능 관련 태스크

| 상황 | 추가 태스크 |
|------|-------------|
| 반복 조회 데이터 | 캐시 Adapter 구현 태스크 |
| 대량 데이터 조회 | 페이징/커서 구현 태스크 |
| N+1 문제 예상 | Fetch Join 쿼리 태스크 |
