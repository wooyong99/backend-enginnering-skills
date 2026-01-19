# 백엔드 개발 핵심 원칙

> **목적**: 유지보수 가능하고 확장 가능한 백엔드 시스템 구축을 위한 기본 원칙  
> **대상**: 백엔드 개발자 및 AI 개발 어시스턴트  
> **철학**: Clean Architecture + Domain-Driven Design 기반

---

## 📋 목차

1. [핵심 아키텍처 원칙](#핵심-아키텍처-원칙)
2. [레이어 분리 원칙](#레이어-분리-원칙)
3. [도메인 모델링 원칙](#도메인-모델링-원칙)
4. [의존성 관리 원칙](#의존성-관리-원칙)
5. [비즈니스 로직 배치 원칙](#비즈니스-로직-배치-원칙)
6. [외부 시스템 통합 원칙](#외부-시스템-통합-원칙)
7. [도메인 간 협력 원칙](#도메인-간-협력-원칙)
8. [명령과 조회 분리 원칙](#명령과-조회-분리-원칙)
9. [코드 작성 가이드라인](#코드-작성-가이드라인)
10. [안티패턴과 해결책](#안티패턴과-해결책)

---

## 핵심 아키텍처 원칙

### 변화의 격리 (Isolation of Change)

**원칙**: 변화가 자주 일어나는 부분과 안정적인 부분을 분리한다.

```
변화 빈도:
높음 → API Layer (인터페이스, 요청/응답 형식)
중간 → Infrastructure Layer (외부 시스템, 기술 스택)
낮음 → Core Layer (비즈니스 규칙, 도메인 로직)
```

**적용 방법**:

- 비즈니스 규칙을 외부 기술로부터 격리
- 인터페이스를 통한 추상화 활용
- 구체적인 구현체는 쉽게 교체 가능하도록 설계

### 의존성 역전 (Dependency Inversion)

**원칙**: 구체적인 것이 추상적인 것에 의존한다.

```
추상적 (안정적)      구체적 (불안정)
    Core       ←─────  Infrastructure
    Core       ←─────  Presentation
    Core       ←─────  Persistence
```

**적용 방법**:

- 핵심 비즈니스 로직은 외부 기술을 모름
- 인터페이스를 통해 의존성 주입
- 외부 레이어가 내부 레이어의 인터페이스를 구현

### 도메인 순수성 (Domain Purity)

**원칙**: 도메인은 외부 세계를 알지 못한다.

**금지사항**:

- 도메인에 프레임워크 어노테이션 사용
- 도메인에 데이터베이스 관련 코드
- 도메인에 HTTP 관련 코드
- 도메인에 외부 라이브러리 직접 의존

**허용사항**:

- 순수한 비즈니스 로직
- 도메인 용어로 표현된 메서드
- 값 객체와 열거형
- 도메인 이벤트

---

## 레이어 분리 원칙

### 레이어 구조

```
┌──────────────────────────────────────┐
│      Presentation Layer              │  ← 사용자 인터페이스
│  (API, Controllers, DTOs)            │
└────────────┬─────────────────────────┘
             │ 호출
             ↓
┌──────────────────────────────────────┐
│         Core Layer                   │  ← 비즈니스 로직
│  ┌────────────────────────────────┐  │
│  │ Application (Use Cases)        │  │
│  │ - 흐름 제어                    │  │
│  │ - 트랜잭션 관리                │  │
│  └────────────────────────────────┘  │
│  ┌────────────────────────────────┐  │
│  │ Domain (Business Rules)        │  │
│  │ - 엔티티                       │  │
│  │ - 비즈니스 규칙                │  │
│  └────────────────────────────────┘  │
└────────────┬─────────────────────────┘
             ↑ 구현
    ┌────────┼────────┐
    │        │        │
┌───┴───┐ ┌─┴────┐ ┌─┴────┐
│Persist│ │Infra │ │Query │  ← 외부 시스템
└───────┘ └──────┘ └──────┘
```

### 1. Presentation Layer (프레젠테이션 레이어)

**역할**:

- 사용자 인터페이스 제공 (REST API, GraphQL, gRPC 등)
- 요청 데이터 검증 및 변환
- 응답 데이터 포맷팅
- 인증/인가 처리

**책임 범위**:

```
✅ 해야 할 것:
- 입력값 형식 검증 (필수값, 타입, 포맷)
- DTO ↔ Command/Result 변환
- Use Case 호출
- HTTP 상태 코드 관리
- 에러 응답 포맷팅

❌ 하지 말아야 할 것:
- 비즈니스 로직 구현
- 데이터베이스 직접 접근
- 상태 변경 로직
- 복잡한 계산 로직
```

**올바른 예시**:

```
Controller/Handler {
    입력 검증
    ↓
    DTO → Command 변환
    ↓
    Use Case 호출
    ↓
    Result → Response 변환
    ↓
    응답 반환
}
```

### 2. Core Layer (핵심 레이어)

**구성**:

- **Application**: Use Case(흐름 제어), Command/Result(입출력), Port(인터페이스)
- **Domain**: Entity(비즈니스 엔티티), Value Object(값 객체), Domain Service(도메인 서비스)

#### Application 서브레이어

**역할**:

- 비즈니스 유스케이스 조율
- 트랜잭션 경계 설정
- 외부 시스템 연동 조율

**구성 요소**:

1. **Use Case**: 비즈니스 흐름 조율

   ```
   Use Case {
       1. 입력 검증
       2. 도메인 로직 호출
       3. 외부 시스템 연동 (Port 사용)
       4. 결과 반환
   }
   ```

2. **Command**: 불변 입력 객체

   ```
   특징:
   - 불변 (immutable)
   - 검증 로직 포함 가능
   - 명확한 의도 표현
   ```

3. **Result**: 불변 출력 객체

   ```
   특징:
   - 불변 (immutable)
   - 필요한 정보만 포함
   - 계층 간 데이터 전달
   ```

4. **Port**: 외부 시스템 인터페이스

   ```
   종류:
   - Repository: 데이터 저장소
   - External API Client: 외부 API
   - Event Publisher: 이벤트 발행
   - Notification: 알림 전송
   ```

5. **Validator**: 복잡한 검증 로직 (선택적)

   ```
   사용 시기:
   - 외부 시스템 데이터 검증 필요
   - 여러 조건의 복합 검증
   - Use Case 코드 복잡도 감소
   ```

6. **Policy**: 복잡한 정책 로직 (선택적)
   ```
   사용 시기:
   - 복잡한 계산 로직
   - 여러 규칙의 조합
   - 정책이 자주 변경됨
   ```

#### Domain 서브레이어

**역할**:

- 핵심 비즈니스 규칙 구현
- 도메인 용어 표현
- 불변성과 일관성 유지

**구성 요소**:

1. **Entity (엔티티)**

   ```
   특징:
   - 고유 식별자 보유
   - 생명주기 존재
   - 비즈니스 로직 포함
   - 상태 변경 메서드 제공
   ```

2. **Value Object (값 객체)**

   ```
   특징:
   - 불변 (immutable)
   - 동등성 비교 (값 기반)
   - 자가 검증
   - 의미 있는 단위 표현
   ```

3. **Domain Service (도메인 서비스)**

   ```
   사용 시기:
   - 여러 엔티티를 조합한 로직
   - 엔티티에 속하지 않는 계산
   - 순수 비즈니스 규칙

   제약사항:
   - 외부 시스템 접근 불가
   - Repository 사용 불가
   - 순수 함수형으로 작성
   ```

4. **Aggregate (집합)**

   ```
   원칙:
   - 트랜잭션 일관성 경계
   - Aggregate Root를 통해서만 접근
   - 작게 유지 (성능 고려)
   ```

5. **Domain Event (도메인 이벤트)**
   ```
   특징:
   - 불변 (immutable)
   - 과거형 명명 (UserRegistered)
   - 도메인 변화 표현
   ```

### 3. Infrastructure Layer (인프라 레이어)

**역할**:

- Core Layer의 Port 구현
- 외부 시스템 연동
- 기술적 세부사항 처리

**구성**:

- **Adapter**: Port 인터페이스 구현체
- **Mapper**: 도메인 ↔ 외부 형식 변환
- **Configuration**: 외부 시스템 설정

**원칙**:

```
✅ 해야 할 것:
- Port 인터페이스 정확히 구현
- 도메인 객체로 변환
- 예외를 도메인 예외로 변환
- 외부 시스템 장애 처리

❌ 하지 말아야 할 것:
- 비즈니스 로직 포함
- Core Layer 객체 직접 수정
- 암묵적 상태 변경
```

### 4. Persistence Layer (영속성 레이어)

**역할**:

- 데이터 저장 및 조회
- Repository 구현
- 데이터베이스 매핑

**원칙**:

```
✅ 해야 할 것:
- 도메인 객체 ↔ 데이터베이스 변환
- 트랜잭션 지원
- 동시성 제어
- 낙관적/비관적 락 관리

❌ 하지 말아야 할 것:
- 비즈니스 로직 구현
- 도메인 객체에 영속성 로직
- 복잡한 조인 쿼리 남발
```

### 5. Query Layer (조회 레이어) - CQRS 적용 시

**역할**:

- 읽기 전용 쿼리 최적화
- 복잡한 조회 처리
- 성능 최적화

**원칙**:

```
✅ 해야 할 것:
- 읽기 전용 쿼리
- 필요한 컬럼만 조회
- 조인 최적화
- 캐싱 활용

❌ 하지 말아야 할 것:
- 상태 변경
- 트랜잭션 사용
- 도메인 로직 구현
```

---

## 도메인 모델링 원칙

### Rich Domain Model (풍부한 도메인 모델)

**원칙**: 데이터와 행위를 함께 배치한다.

**올바른 예시**:

```
Entity {
    private Status status;
    private List<Item> items;

    // ✅ 비즈니스 메서드
    public void complete() {
        validateCanComplete();
        this.status = Status.COMPLETED;
        this.completedAt = now();
        raiseEvent(new CompletedEvent());
    }

    private void validateCanComplete() {
        if (this.status != Status.PAID) {
            throw new InvalidStatusException();
        }
        if (this.items.isEmpty()) {
            throw new EmptyItemsException();
        }
    }
}
```

**잘못된 예시** (Anemic Domain Model):

```
Entity {
    private Status status;

    // ❌ Getter/Setter만 존재
    public void setStatus(Status status) {
        this.status = status;
    }
}

// 비즈니스 로직이 서비스로 누출
Service {
    public void complete(Entity entity) {
        if (entity.getStatus() != Status.PAID) {
            throw new InvalidStatusException();
        }
        entity.setStatus(Status.COMPLETED);
    }
}
```

### 불변성 (Immutability)

**원칙**: 가능한 모든 것을 불변으로 만든다.

**불변 객체 대상**:

- Value Object (항상)
- Command/Result (항상)
- Domain Event (항상)
- Entity의 일부 속성 (가능한 경우)

**이점**:

- 스레드 안전성
- 예측 가능한 동작
- 디버깅 용이
- 캐싱 안전

### 값 객체 활용 (Value Object)

**원칙**: 의미 있는 개념을 값 객체로 표현한다.

**대상**:

```
❌ 원시 타입 남발:
String email
String phoneNumber
BigDecimal amount
String currency

✅ 값 객체 활용:
Email email
PhoneNumber phoneNumber
Money amount
```

**값 객체 특징**:

```
Value Object {
    - 불변성
    - 자가 검증
    - 동등성 비교
    - 풍부한 메서드
}

예시:
Money {
    private BigDecimal amount;
    private Currency currency;

    public Money add(Money other) {
        validateSameCurrency(other);
        return new Money(
            this.amount.add(other.amount),
            this.currency
        );
    }
}
```

### 정적 팩토리 메서드 (Static Factory Method)

**원칙**: 생성자 대신 의도를 드러내는 정적 메서드를 사용한다.

**올바른 예시**:

```
Entity {
    // ✅ 의도가 명확한 정적 팩토리 메서드
    public static Entity create(Command command) {
        validate(command);
        return new Entity(...);
    }

    public static Entity createForTest(Data data) {
        return new Entity(...);
    }

    public static Entity reconstruct(Data data) {
        // 검증 없이 재구성 (저장소에서 로딩)
        return new Entity(...);
    }

    // 생성자는 protected/private
    protected Entity() {}
}
```

### 비즈니스 메서드 명명 (Business Method Naming)

**원칙**: 도메인 용어를 사용하여 메서드를 명명한다.

```
✅ 도메인 언어:
order.place()          // 주문하다
order.cancel()         // 취소하다
payment.approve()      // 승인하다
user.register()        // 등록하다
voucher.issue()        // 발급하다

❌ 기술 용어:
order.updateStatus()
order.setStatus()
order.changeState()
```

---

## 의존성 관리 원칙

### 단방향 의존성 (Unidirectional Dependency)

**원칙**: 의존성은 항상 한 방향으로만 흐른다.

```
올바른 의존성:
Presentation → Core
Infrastructure → Core
Persistence → Core
Query → Core

금지된 의존성:
Core → Infrastructure ❌
Core → Persistence ❌
Domain → Application ❌
```

### 인터페이스 기반 의존성 (Interface-based Dependency)

**원칙**: 구체 클래스가 아닌 인터페이스에 의존한다.

**올바른 예시**:

```
Use Case {
    // ✅ 인터페이스에 의존
    private final EntityRepository repository;
    private final ExternalClient client;
    private final EventPublisher eventPublisher;
}
```

**잘못된 예시**:

```
Use Case {
    // ❌ 구체 클래스에 의존
    private final JpaEntityRepository repository;
    private final RestTemplateClient client;
    private final KafkaEventPublisher eventPublisher;
}
```

### 순환 의존성 금지 (No Circular Dependency)

**원칙**: 모듈/패키지 간 순환 의존성을 허용하지 않는다.

**문제 상황**:

```
❌ 순환 의존:
Order Use Case → Payment Repository
Payment Use Case → Order Repository
```

**해결 방법**:

```
✅ 이벤트로 분리:
Order Use Case → Event Publisher → Order Created Event
Payment Event Handler ← Order Created Event
```

---

## 비즈니스 로직 배치 원칙

### 로직 배치 결정 트리

```
┌─────────────────────────────┐
│ 비즈니스 로직인가?          │
└────────┬────────────────────┘
         │
    ┌────┴────┐
    │         │
   Yes       No (기술적 관심사)
    │         └─→ Infrastructure
    ▼
┌─────────────────────────────┐
│ 단일 엔티티 규칙인가?       │
└────────┬────────────────────┘
         │
    ┌────┴────┐
    │         │
   Yes       No
    │         │
    ▼         ▼
 Entity   ┌─────────────────────────┐
 Method   │ 외부 의존성 필요한가?   │
          └────────┬────────────────┘
                   │
              ┌────┴────┐
              │         │
             Yes       No
              │         │
              ▼         ▼
         Application  Domain
         Validator/   Service
         Policy
```

### Entity vs Domain Service vs Application Service

| 기준        | Entity      | Domain Service | Application Service |
| ----------- | ----------- | -------------- | ------------------- |
| 로직 범위   | 단일 엔티티 | 여러 엔티티    | 유스케이스 흐름     |
| 외부 의존성 | 불가        | 불가           | 가능 (Port)         |
| 트랜잭션    | 없음        | 없음           | 있음                |
| 상태 변경   | 자신만      | 불가           | 가능                |

**Entity 예시**:

```
Order {
    public void addItem(Item item) {
        // 단일 엔티티 규칙
        validateCanAddItem(item);
        this.items.add(item);
        recalculateTotal();
    }
}
```

**Domain Service 예시**:

```
PriceCalculator {
    // 여러 엔티티 조합, 외부 의존 없음
    public Money calculate(Order order, List<Coupon> coupons) {
        Money itemTotal = order.calculateItemTotal();
        Money discount = calculateDiscount(itemTotal, coupons);
        Money shipping = calculateShipping(order);
        return itemTotal.subtract(discount).add(shipping);
    }
}
```

**Application Service 예시**:

```
CreateOrderUseCase {
    // 유스케이스 흐름, 외부 의존 허용
    public Result execute(Command command) {
        validateInventory(command);  // Port 사용

        Order order = Order.create(command);
        Money price = priceCalculator.calculate(order);  // Domain Service
        order.setPrice(price);

        repository.save(order);  // Port 사용
        eventPublisher.publish(event);  // Port 사용

        return Result.from(order);
    }
}
```

---

## 외부 시스템 통합 원칙

### Port-Adapter 패턴

**원칙**: 외부 시스템은 Port 인터페이스를 통해서만 접근한다.

**구조**:

```
Core (Port Interface)
    ↑
    │ implements
    │
Infrastructure (Adapter)
    ↓
External System
```

**Port 정의 예시**:

```
// Core Layer
interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
    PaymentStatus checkStatus(String transactionId);
    void cancel(String transactionId);
}
```

**Adapter 구현 예시**:

```
// Infrastructure Layer
class ExternalPaymentAdapter implements PaymentProcessor {
    private final ExternalApiClient client;
    private final PaymentMapper mapper;

    public PaymentResult process(PaymentRequest request) {
        // 1. 도메인 → 외부 API 형식 변환
        ApiRequest apiRequest = mapper.toApiRequest(request);

        // 2. 외부 API 호출
        ApiResponse apiResponse = client.call(apiRequest);

        // 3. 외부 API → 도메인 형식 변환
        return mapper.toDomainResult(apiResponse);
    }
}
```

### 예외 변환 (Exception Translation)

**원칙**: 외부 시스템 예외를 도메인 예외로 변환한다.

**올바른 예시**:

```
Adapter {
    public Result process(Request request) {
        try {
            return externalClient.call(request);
        } catch (ExternalApiException e) {
            // ✅ 도메인 예외로 변환
            throw new ExternalServiceUnavailableException(
                "결제 서비스 일시적 오류", e
            );
        } catch (TimeoutException e) {
            throw new ExternalServiceTimeoutException(
                "결제 서비스 응답 시간 초과", e
            );
        }
    }
}
```

### 재시도 및 장애 처리 (Retry and Resilience)

**원칙**: 외부 시스템 장애에 대비한 전략을 수립한다.

**전략**:

```
1. Retry (재시도)
   - 일시적 오류 대응
   - Exponential Backoff
   - 최대 재시도 횟수 제한

2. Circuit Breaker (회로 차단기)
   - 연속 실패 시 빠른 실패
   - 일정 시간 후 재시도

3. Timeout (시간 제한)
   - 무한 대기 방지
   - 적절한 타임아웃 설정

4. Fallback (대체 방안)
   - 기본값 반환
   - 캐시된 데이터 사용
```

---

## 도메인 간 협력 원칙

### 도메인 간 의존성 관리

**원칙**: 도메인 간 결합도를 최소화한다.

**허용되는 의존성**:

```
✅ 조회 (Read):
- 다른 도메인의 Repository 직접 호출 가능
- Use Case에서 여러 도메인 Repository 사용 가능

예시:
OrderUseCase {
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;  // ✅ OK

    public void execute(Command command) {
        User user = userRepository.findById(command.getUserId());
        Order order = Order.create(command, user);
    }
}
```

**금지되는 의존성**:

```
❌ 상태 변경 (Write):
- 다른 도메인의 엔티티 상태 직접 변경 금지

잘못된 예시:
PaymentUseCase {
    private final VoucherRepository voucherRepository;

    public void execute(Command command) {
        payment.complete();

        // ❌ 다른 도메인 상태 변경
        Voucher voucher = voucherRepository.find(...);
        voucher.issue();  // ❌ 금지
    }
}
```

### 이벤트 기반 협력 (Event-Driven Collaboration)

**원칙**: 도메인 간 상태 변경은 이벤트를 통해 전파한다.

**패턴**:

```
1. 이벤트 발행
   Domain A Use Case
   ↓
   Domain Event 발행

2. 이벤트 구독
   Domain Event
   ↓
   Domain B Event Handler
   ↓
   Domain B Use Case 실행
```

**구현 예시**:

```
// 발행 측
PaymentUseCase {
    private final EventPublisher eventPublisher;

    public void execute(Command command) {
        Payment payment = payment.complete();
        repository.save(payment);

        // ✅ 이벤트 발행
        eventPublisher.publish(
            new PaymentCompletedEvent(
                payment.getId(),
                payment.getOrderId(),
                payment.getAmount()
            )
        );
    }
}

// 구독 측
VoucherEventHandler {
    private final IssueVoucherUseCase issueVoucherUseCase;

    @EventListener
    public void handle(PaymentCompletedEvent event) {
        issueVoucherUseCase.execute(
            new IssueVoucherCommand(event.getOrderId())
        );
    }
}
```

### 트랜잭션 경계 (Transaction Boundary)

**원칙**: 하나의 Use Case는 하나의 Aggregate만 수정한다.

**올바른 예시**:

```
✅ 단일 Aggregate 수정:
CreateOrderUseCase {
    @Transactional
    public void execute(Command command) {
        Order order = Order.create(command);
        repository.save(order);  // Order만 수정
    }
}
```

**잘못된 예시**:

```
❌ 여러 Aggregate 수정:
CompleteOrderUseCase {
    @Transactional
    public void execute(Command command) {
        order.complete();
        orderRepository.save(order);

        // ❌ 같은 트랜잭션에서 다른 Aggregate 수정
        voucher.issue();
        voucherRepository.save(voucher);
    }
}
```

**해결책**:

```
✅ 이벤트로 분리:
CompleteOrderUseCase {
    @Transactional
    public void execute(Command command) {
        order.complete();
        orderRepository.save(order);
        eventPublisher.publish(new OrderCompletedEvent());
    }
}

VoucherEventHandler {
    @Transactional
    public void handle(OrderCompletedEvent event) {
        voucher.issue();
        voucherRepository.save(voucher);
    }
}
```

---

## 명령과 조회 분리 원칙 (CQRS)

### Command와 Query 분리

**원칙**: 상태를 변경하는 명령과 데이터를 조회하는 쿼리를 분리한다.

**Command (명령)**:

```
특징:
- 상태 변경
- 트랜잭션 필요
- 비즈니스 로직 포함
- 도메인 모델 사용
- 반환값 최소화

구조:
Use Case
  ↓
Domain Entity
  ↓
Repository (Write)
```

**Query (조회)**:

```
특징:
- 읽기 전용
- 트랜잭션 불필요
- 성능 최적화
- DTO 직접 조회
- 필요한 데이터만 조회

구조:
Query Service
  ↓
Optimized Query (SQL, NoSQL)
  ↓
DTO 반환
```

### Query 최적화 전략

**원칙**: 조회는 성능을 최우선으로 최적화한다.

**최적화 기법**:

```
1. 필요한 컬럼만 선택
   SELECT id, name, status
   (SELECT * 지양)

2. 조인 최소화
   - N+1 문제 해결
   - Fetch Join 활용
   - Batch Size 설정

3. 인덱스 활용
   - WHERE, ORDER BY 컬럼 인덱스
   - 복합 인덱스 고려

4. 페이징 처리
   - Offset/Limit
   - Cursor-based Pagination

5. 캐싱
   - 자주 조회되는 데이터
   - 변경 빈도 낮은 데이터
```

---

## 코드 작성 가이드라인

### 명명 규칙 (Naming Convention)

**Use Case**:

```
패턴: {Action}{Entity}UseCase
예시:
- CreateOrderUseCase
- ProcessPaymentUseCase
- CancelSubscriptionUseCase
```

**Command**:

```
패턴: {Action}Command 또는 {Action}{Entity}Command
예시:
- CreateOrderCommand
- PaymentCommand
- RegisterUserCommand
```

**Result**:

```
패턴: {Entity}Result 또는 {Action}Result
예시:
- OrderResult
- PaymentResult
- LoginResult
```

**Port (Repository)**:

```
패턴: {Entity}Repository
예시:
- OrderRepository
- UserRepository
- ProductRepository
```

**Port (External)**:

```
패턴: {Purpose}{Provider} 또는 {Action}{Target}
예시:
- PaymentProcessor
- SmsNotifier
- EmailSender
- InventoryChecker
```

**Domain Service**:

```
패턴: {Purpose}Calculator/Manager/Policy
예시:
- PriceCalculator
- DiscountPolicy
- ShippingFeeCalculator
```

**Entity**:

```
패턴: 명사 단수형
예시:
- Order
- User
- Product
- Payment
```

**Value Object**:

```
패턴: 명사형
예시:
- Money
- Email
- PhoneNumber
- Address
```

**Event**:

```
패턴: {Entity}{Action}Event (과거형)
예시:
- OrderCreatedEvent
- PaymentCompletedEvent
- UserRegisteredEvent
```

### 메서드 명명 (Method Naming)

**Use Case 메서드**:

```
패턴: execute(Command) → Result
또는: handle(Command) → Result

예시:
public OrderResult execute(CreateOrderCommand command)
public void execute(CancelOrderCommand command)
```

**도메인 메서드**:

```
✅ 도메인 언어 사용:
- order.place()
- payment.approve()
- user.register()
- voucher.issue()

❌ 기술 용어 사용:
- order.updateStatus()
- payment.setApproved()
- user.insertRecord()
```

**Repository 메서드**:

```
패턴:
- find{Entity}By{Condition}
- save{Entity}
- delete{Entity}
- exists{Entity}By{Condition}

예시:
- findOrderById(String id)
- findOrdersByUserId(String userId)
- saveOrder(Order order)
- deleteOrder(String id)
- existsOrderByIdAndStatus(String id, Status status)
```

### 주석 작성 (Documentation)

**Port 인터페이스**:

```
/**
 * 주문 저장소 포트
 *
 * 주문 엔티티의 영속성을 담당합니다.
 */
interface OrderRepository {

    /**
     * 주문을 저장합니다.
     *
     * @param order 저장할 주문
     * @return 저장된 주문
     * @throws OrderAlreadyExistsException 동일 ID의 주문이 이미 존재하는 경우
     */
    Order save(Order order);

    /**
     * ID로 주문을 조회합니다.
     *
     * @param id 주문 ID
     * @return 조회된 주문
     * @throws OrderNotFoundException 주문을 찾을 수 없는 경우
     */
    Order findById(String id);
}
```

**Use Case**:

```
/**
 * 주문 생성 유스케이스
 *
 * 사용자의 장바구니를 기반으로 주문을 생성합니다.
 * - 재고 확인
 * - 가격 계산
 * - 주문 저장
 * - 주문 생성 이벤트 발행
 */
class CreateOrderUseCase {
    /**
     * 주문을 생성합니다.
     *
     * @param command 주문 생성 명령
     * @return 생성된 주문 정보
     * @throws InsufficientInventoryException 재고 부족
     * @throws InvalidPriceException 가격 계산 오류
     */
    public OrderResult execute(CreateOrderCommand command) {
        // 구현
    }
}
```

### 예외 처리 (Exception Handling)

**도메인 예외**:

```
// 비즈니스 규칙 위반
class InvalidOrderStatusException extends DomainException {
    public InvalidOrderStatusException(String message) {
        super(message);
    }
}

// 엔티티 미존재
class OrderNotFoundException extends DomainException {
    public OrderNotFoundException(String orderId) {
        super("Order not found: " + orderId);
    }
}
```

**예외 계층**:

```
Exception
  ↓
RuntimeException
  ↓
DomainException (도메인 예외 base)
  ├─ ValidationException (검증 실패)
  ├─ BusinessRuleViolationException (규칙 위반)
  ├─ ResourceNotFoundException (리소스 미존재)
  └─ ExternalServiceException (외부 서비스 오류)
```

---

## 안티패턴과 해결책

### 안티패턴 1: 비즈니스 로직이 Presentation에 위치

**문제**:

```
Controller {
    public Response createOrder(Request request) {
        // ❌ 비즈니스 로직이 Controller에
        if (user.getGrade() == Grade.VIP) {
            order.applyDiscount(0.1);
        }

        if (order.getTotalAmount() > 50000) {
            order.setShippingFee(0);
        }

        orderRepository.save(order);
    }
}
```

**해결책**:

```
Controller {
    public Response createOrder(Request request) {
        // ✅ Use Case에 위임
        Command command = request.toCommand();
        Result result = createOrderUseCase.execute(command);
        return Response.from(result);
    }
}

CreateOrderUseCase {
    public Result execute(Command command) {
        Order order = Order.create(command);
        order.applyDiscountPolicy(discountPolicy);
        order.calculateShipping();
        repository.save(order);
        return Result.from(order);
    }
}
```

### 안티패턴 2: Anemic Domain Model

**문제**:

```
// ❌ 데이터만 있는 도메인
Order {
    private String id;
    private OrderStatus status;

    // Getter/Setter만
    public void setStatus(OrderStatus status) {
        this.status = status;
    }
}

// 로직이 서비스로 누출
OrderService {
    public void complete(String orderId) {
        Order order = repository.findById(orderId);

        // ❌ 비즈니스 규칙이 서비스에
        if (order.getStatus() != OrderStatus.PAID) {
            throw new Exception();
        }

        order.setStatus(OrderStatus.COMPLETED);
        repository.save(order);
    }
}
```

**해결책**:

```
// ✅ Rich Domain Model
Order {
    private String id;
    private OrderStatus status;

    // 비즈니스 메서드
    public void complete() {
        if (this.status != OrderStatus.PAID) {
            throw new InvalidOrderStatusException(
                "결제되지 않은 주문은 완료할 수 없습니다"
            );
        }

        this.status = OrderStatus.COMPLETED;
        this.completedAt = LocalDateTime.now();
    }

    // Setter 제거
}

OrderUseCase {
    public void execute(Command command) {
        Order order = repository.findById(command.getOrderId());
        order.complete();  // ✅ 도메인 메서드 호출
        repository.save(order);
    }
}
```

### 안티패턴 3: 도메인이 Infrastructure에 의존

**문제**:

```
// ❌ 도메인이 JPA에 의존
Domain/Order {
    @Entity  // ❌ JPA 어노테이션
    @Table(name = "orders")
    class Order {
        @Id
        @GeneratedValue
        private Long id;

        @Column(name = "order_status")
        private String status;
    }
}
```

**해결책**:

```
// ✅ 순수 도메인
Domain/Order {
    class Order {
        private String id;
        private OrderStatus status;

        // 순수 비즈니스 로직만
    }
}

// 영속성은 Persistence Layer에서 처리
Persistence/OrderEntity {
    @Entity
    @Table(name = "orders")
    class OrderEntity {
        @Id
        private String id;

        @Column(name = "order_status")
        private String status;

        // Domain ↔ Entity 변환
        public Order toDomain() {
            return Order.builder()
                .id(this.id)
                .status(OrderStatus.valueOf(this.status))
                .build();
        }

        public static OrderEntity from(Order order) {
            OrderEntity entity = new OrderEntity();
            entity.id = order.getId();
            entity.status = order.getStatus().name();
            return entity;
        }
    }
}
```

### 안티패턴 4: God Class

**문제**:

```
// ❌ 모든 것을 하는 거대한 Use Case
CreateOrderUseCase {
    public Result execute(Command command) {
        // 100+ 줄의 복잡한 로직
        // - 사용자 검증 30줄
        // - 재고 확인 20줄
        // - 가격 계산 40줄
        // - 쿠폰 적용 30줄
        // - 결제 처리 20줄
        // ...
    }
}
```

**해결책**:

```
// ✅ Validator, Policy로 책임 분리
CreateOrderUseCase {
    private final OrderValidator validator;
    private final PricingPolicy pricingPolicy;
    private final PaymentProcessor paymentProcessor;

    public Result execute(Command command) {
        // 1. 검증 (Validator에 위임)
        validator.validateForCreate(command);

        // 2. 가격 계산 (Policy에 위임)
        Money price = pricingPolicy.calculateTotalPrice(command);

        // 3. 주문 생성 (Domain)
        Order order = Order.create(command, price);

        // 4. 결제 처리 (Port)
        paymentProcessor.process(order);

        // 5. 저장
        repository.save(order);

        return Result.from(order);
    }
}

OrderValidator {
    public void validateForCreate(Command command) {
        // 검증 로직 집중
    }
}

PricingPolicy {
    public Money calculateTotalPrice(Command command) {
        // 가격 계산 로직 집중
    }
}
```

### 안티패턴 5: 도메인 간 순환 의존

**문제**:

```
// ❌ 순환 의존
Order Use Case {
    private final PaymentRepository paymentRepository;  // Order → Payment
}

Payment Use Case {
    private final OrderRepository orderRepository;  // Payment → Order
}
```

**해결책**:

```
// ✅ 이벤트로 결합도 제거
Order Use Case {
    private final EventPublisher eventPublisher;

    public void execute(Command command) {
        Order order = Order.create(command);
        repository.save(order);

        // 이벤트 발행
        eventPublisher.publish(
            new OrderCreatedEvent(order.getId(), order.getAmount())
        );
    }
}

Payment Event Handler {
    private final CreatePaymentUseCase createPaymentUseCase;

    @EventListener
    public void handle(OrderCreatedEvent event) {
        createPaymentUseCase.execute(
            new CreatePaymentCommand(event.getOrderId(), event.getAmount())
        );
    }
}
```

### 안티패턴 6: 트랜잭션 경계 불분명

**문제**:

```
// ❌ 트랜잭션 경계가 모호함
Service {
    public void process() {
        // 여러 Aggregate 수정
        order.complete();
        orderRepository.save(order);

        payment.process();
        paymentRepository.save(payment);

        voucher.issue();
        voucherRepository.save(voucher);

        // 중간에 실패하면?
    }
}
```

**해결책**:

```
// ✅ 명확한 트랜잭션 경계
Order Use Case {
    @Transactional
    public void execute(Command command) {
        order.complete();
        repository.save(order);
        eventPublisher.publish(new OrderCompletedEvent());
        // 하나의 Aggregate만 수정
    }
}

Payment Event Handler {
    @Transactional
    public void handle(OrderCompletedEvent event) {
        payment.process();
        repository.save(payment);
        // 별도 트랜잭션
    }
}

Voucher Event Handler {
    @Transactional
    public void handle(OrderCompletedEvent event) {
        voucher.issue();
        repository.save(voucher);
        // 별도 트랜잭션
    }
}
```

---

## 체크리스트

### 신규 기능 개발 시

#### 설계 단계

- [ ] 비즈니스 요구사항을 명확히 이해했는가?
- [ ] 어느 도메인에 속하는가?
- [ ] 명령(Command)인가 조회(Query)인가?
- [ ] 다른 도메인과의 관계는 무엇인가?

#### 구현 단계

- [ ] 비즈니스 로직이 도메인 레이어에 있는가?
- [ ] Use Case는 흐름 제어만 하는가?
- [ ] Port 인터페이스로 외부 의존성을 추상화했는가?
- [ ] 명명 규칙을 준수했는가?

#### 검증 단계

- [ ] 의존성 방향이 올바른가? (단방향)
- [ ] 순환 의존성이 없는가?
- [ ] 도메인이 외부 기술에 의존하지 않는가?
- [ ] 테스트 코드를 작성했는가?

### 코드 리뷰 시

#### 아키텍처

- [ ] 레이어 분리가 명확한가?
- [ ] 의존성 방향이 올바른가?
- [ ] 각 레이어의 책임이 명확한가?

#### 도메인 모델

- [ ] Rich Domain Model인가? (Anemic이 아닌가?)
- [ ] 불변성을 적절히 활용하는가?
- [ ] 값 객체를 사용하는가?
- [ ] 도메인 용어를 사용하는가?

#### Use Case

- [ ] Use Case가 간결한가? (50줄 이하)
- [ ] Port만 의존하는가?
- [ ] 트랜잭션 경계가 명확한가?
- [ ] 하나의 책임만 가지는가?

#### 외부 통합

- [ ] Port-Adapter 패턴을 사용하는가?
- [ ] 예외를 도메인 예외로 변환하는가?
- [ ] 재시도 및 장애 처리가 적절한가?

---

## 마무리

### 핵심 원칙 요약

1. **변화의 격리**: 비즈니스 로직을 외부 기술로부터 격리
2. **의존성 역전**: 구체적인 것이 추상적인 것에 의존
3. **도메인 순수성**: 도메인은 외부 세계를 모름
4. **단방향 의존성**: 의존성은 한 방향으로만
5. **Rich Domain Model**: 데이터와 행위를 함께
6. **명확한 책임**: 각 레이어와 객체는 하나의 책임만
7. **이벤트 기반 협력**: 도메인 간 느슨한 결합
8. **명령과 조회 분리**: 상태 변경과 데이터 조회 분리

### 지속적 개선

이 원칙들은:

- 절대적인 규칙이 아닌 가이드라인입니다
- 프로젝트 상황에 맞게 조정 가능합니다
- 팀의 합의를 통해 발전시킵니다
- 지속적으로 개선하고 업데이트합니다

**"완벽한 아키텍처는 없습니다. 프로젝트와 팀에 맞는 적절한 아키텍처가 있을 뿐입니다."**
