# 아키텍처 의사결정 프레임워크

## 목적

코드 작성 시 아키텍처 관점의 의사결정을 체계적으로 내리기 위한 프레임워크.

---

## 의사결정 트리

### 1. 레이어 배치 결정

**질문: "이 코드는 왜 변경되는가?"**

```
변경 이유 분석
├─ 비즈니스 규칙 변경
│  └─ Domain Layer
│     ├─ Entity: 식별자 있는 비즈니스 객체
│     ├─ Value Object: 불변 값 객체
│     ├─ Domain Service: 여러 엔티티 조합 로직
│     └─ Policy: 비즈니스 정책 객체
│
├─ 비즈니스 플로우/유스케이스 변경
│  └─ Application Layer
│     ├─ Use Case: 단일 비즈니스 플로우
│     ├─ Command/Result: 입출력 DTO
│     └─ Port: 외부 의존성 인터페이스
│
├─ 외부 시스템/프레임워크 변경
│  └─ Infrastructure Layer
│     ├─ Adapter: Port 구현체
│     ├─ Repository Impl: DB 접근
│     └─ ApiClient: 외부 API 호출
│
└─ UI/프로토콜 변경
   └─ Presentation Layer
      ├─ Controller: HTTP 엔드포인트
      └─ DTO: API 요청/응답 구조
```

**판단 기준:**

```java
// ✅ 올바른 배치
// Domain: 비즈니스 규칙
class Order {
    public void place() {
        if (items.isEmpty()) {
            throw new EmptyOrderException();
        }
        this.status = OrderStatus.PLACED;
    }
}

// Application: 플로우 제어
class PlaceOrderUseCase {
    public Result execute(Command command) {
        Order order = orderRepository.findById(command.getOrderId());
        order.place();
        orderRepository.save(order);
        return Result.success();
    }
}

// Infrastructure: 기술 구현
class OrderJpaRepository implements OrderRepository {
    public Order findById(OrderId id) {
        return jpaRepository.findById(id.getValue())
            .map(this::toDomain)
            .orElseThrow();
    }
}

// ❌ 잘못된 배치
// Domain에 기술 의존성
class Order {
    @Entity  // JPA 의존
    @Table(name = "orders")
    public class Order { }
}

// Use Case에 비즈니스 규칙
class PlaceOrderUseCase {
    public Result execute(Command command) {
        if (order.getItems().isEmpty()) {  // 비즈니스 규칙이 Use Case에
            throw new EmptyOrderException();
        }
    }
}
```

---

### 2. 의존성 방향 결정

**원칙: 의존성은 항상 안정된 방향으로 흐른다**

```
불안정 (자주 변경)           안정 (거의 불변)
┌─────────────────────────────────────────┐
│                                         │
Presentation → Application → Domain ← Infrastructure
    (UI)         (Flow)      (Rules)     (Tech)
                                ↑
                                │
                            Port Interface
```

**의존성 역전 원칙 (DIP)**

```java
// ✅ 올바른 의존성 방향
// Application Layer
interface OrderRepository {  // Port
    Order findById(OrderId id);
    void save(Order order);
}

class PlaceOrderUseCase {
    private final OrderRepository repository;  // 인터페이스 의존

    public PlaceOrderUseCase(OrderRepository repository) {
        this.repository = repository;
    }
}

// Infrastructure Layer
class OrderJpaRepository implements OrderRepository {  // Port 구현
    // JPA 구현 세부사항
}

// ❌ 잘못된 의존성 방향
// Application이 Infrastructure 구체 클래스를 직접 의존
class PlaceOrderUseCase {
    private final OrderJpaRepository repository;  // 구체 클래스 의존
}
```

**판단 체크리스트:**

1. **안정성 순서**: Domain > Application > Infrastructure = Presentation
2. **인터페이스 위치**: 사용하는 쪽(Application)에 정의
3. **구현체 위치**: 기술 세부사항이 있는 쪽(Infrastructure)에 정의
4. **의존성 흐름**: 항상 안정된 방향으로 (→ Domain)

---

### 3. 컴포넌트 경계 결정

**질문: "어떤 것들이 함께 변경되는가?"**

**Common Closure Principle (CCP)**: 함께 변경되는 것은 함께 묶는다

```
주문 컨텍스트
├─ domain/
│  ├─ Order.java
│  ├─ OrderItem.java
│  ├─ OrderStatus.java
│  └─ OrderPolicy.java
├─ application/
│  ├─ PlaceOrderUseCase.java
│  ├─ CancelOrderUseCase.java
│  └─ port/
│     └─ OrderRepository.java
└─ infrastructure/
   └─ OrderJpaRepository.java

결제 컨텍스트 (별도 컴포넌트)
├─ domain/
│  ├─ Payment.java
│  └─ PaymentMethod.java
└─ application/
   └─ ProcessPaymentUseCase.java
```

**Common Reuse Principle (CRP)**: 함께 사용되지 않는 것은 분리한다

```java
// ✅ 응집도 높은 컴포넌트
// 주문 관련 클래스들만 묶음
com.shop.order
├─ domain
│  ├─ Order
│  ├─ OrderItem
│  └─ OrderPolicy
└─ application
   └─ PlaceOrderUseCase

// ❌ 응집도 낮은 컴포넌트
// 관련 없는 클래스들이 섞임
com.shop.core
├─ Order
├─ Payment
├─ User
└─ Product  // 너무 많은 책임
```

**Acyclic Dependencies Principle (ADP)**: 순환 의존성 제거

```java
// ❌ 순환 의존성
Order → Payment → Order  // 순환!

// ✅ 이벤트로 순환 제거
Order → OrderPlacedEvent → Payment
```

---

### 4. 책임 분배 결정

**질문: "누가 이 작업을 수행하기에 가장 적합한가?"**

**Information Expert**: 정보를 가진 객체가 책임진다

```java
// ✅ 정보를 가진 Order가 계산
class Order {
    private List<OrderItem> items;

    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// ❌ 외부에서 Order 정보를 꺼내서 계산
class OrderService {
    public Money calculateTotal(Order order) {
        Money total = Money.ZERO;
        for (OrderItem item : order.getItems()) {  // Tell, Don't Ask 위반
            total = total.add(item.getSubtotal());
        }
        return total;
    }
}
```

**Single Responsibility Principle (SRP)**: 변경 이유는 하나만

```java
// ✅ 단일 책임
class OrderValidator {
    public void validate(Order order) {
        // 검증만 책임
    }
}

class OrderNotifier {
    public void notifyPlaced(Order order) {
        // 알림만 책임
    }
}

// ❌ 여러 책임
class OrderService {
    public void placeOrder(Order order) {
        // 검증
        validate(order);
        // 저장
        save(order);
        // 알림
        sendEmail(order);
        // 결제
        processPayment(order);
    }
}
```

**Low Coupling**: 의존성 최소화

```java
// ✅ 낮은 결합도 - 인터페이스 의존
class PlaceOrderUseCase {
    private final OrderRepository repository;  // 인터페이스
    private final PaymentGateway gateway;      // 인터페이스
}

// ❌ 높은 결합도 - 구체 클래스 의존
class PlaceOrderUseCase {
    private final OrderJpaRepository repository;     // 구체 클래스
    private final StripePaymentClient gateway;       // 구체 클래스
    private final SendGridEmailClient emailClient;   // 구체 클래스
}
```

**High Cohesion**: 관련된 것끼리 묶기

```java
// ✅ 높은 응집도
class Order {
    // 주문 관련 데이터
    private OrderId id;
    private List<OrderItem> items;
    private Money totalAmount;

    // 주문 관련 행위
    public void place() { }
    public void cancel() { }
    public Money calculateTotal() { }
}

// ❌ 낮은 응집도
class Order {
    // 주문 데이터
    private OrderId id;
    private List<OrderItem> items;

    // 결제 데이터 (다른 책임)
    private PaymentMethod paymentMethod;
    private String paymentTransactionId;

    // 배송 데이터 (다른 책임)
    private Address shippingAddress;
    private Carrier carrier;
}
```

---

## 자동 판단 알고리즘

### 레이어 자동 결정

```
입력: 코드 또는 기능 설명

1. 키워드 분석
   - "규칙", "검증", "계산", "정책" → Domain
   - "흐름", "조회", "처리", "실행" → Application
   - "저장", "API 호출", "메시지 발송" → Infrastructure
   - "요청", "응답", "엔드포인트" → Presentation

2. 의존성 분석
   - 기술 라이브러리 사용? → Infrastructure
   - 비즈니스 객체만 사용? → Domain
   - Repository/Port 사용? → Application

3. 변경 빈도 분석
   - 비즈니스 요구사항 변경 시? → Domain
   - 플로우 변경 시? → Application
   - 기술 스택 변경 시? → Infrastructure
```

### 의존성 방향 자동 검증

```
입력: Class A → Class B 의존성

1. 레이어 위치 확인
   A_layer = getLayer(A)
   B_layer = getLayer(B)

2. 의존성 방향 검증
   if A_layer의 안정성 < B_layer의 안정성:
       return "올바른 의존성"
   else:
       return "의존성 역전 필요"

3. 안정성 순서
   Domain(3) > Application(2) > Infrastructure(1) = Presentation(1)
```

### 컴포넌트 경계 자동 제안

```
입력: 클래스 집합

1. 변경 상관관계 분석
   for each pair (ClassA, ClassB):
       if ClassA와 ClassB가 함께 변경된 이력 > 임계값:
           같은 컴포넌트로 제안

2. 사용 관계 분석
   for each ClassA:
       used_by = ClassA를 사용하는 클래스들
       if len(used_by) > 임계값:
           ClassA는 별도 공유 컴포넌트로 분리 제안

3. 순환 의존성 검사
   if 순환 의존성 발견:
       이벤트 또는 인터페이스로 순환 제거 제안
```

### 책임 자동 분배

```
입력: 기능 또는 메서드

1. 정보 소유자 확인
   needed_data = 기능이 필요로 하는 데이터
   owner = needed_data를 소유한 클래스
   return "owner가 이 책임을 가져야 함"

2. 책임 크기 검증
   if 클래스의 메서드 수 > 10:
       return "책임 분리 필요 - SRP 위반 가능성"

3. 결합도 검증
   dependencies = 클래스의 의존성 개수
   if dependencies > 5:
       return "결합도 높음 - 인터페이스 도입 고려"
```

---

## 실전 적용 예시

### 예시 1: 새 기능 구현

**요청**: "사용자가 주문을 취소할 수 있게 해주세요"

**의사결정 프로세스**:

```
1. 레이어 결정
   - "주문 취소" → 비즈니스 규칙 → Domain Layer
   - 취소 플로우 제어 → Application Layer

2. 책임 분배
   - 취소 가능 여부 검증 → Order 엔티티 (Information Expert)
   - 취소 처리 흐름 → CancelOrderUseCase
   - 환불 처리 → PaymentService (외부 Port)

3. 구현
```

```java
// Domain Layer
class Order {
    public void cancel() {
        validateCanCancel();  // 비즈니스 규칙
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(this.id));
    }

    private void validateCanCancel() {
        if (this.status == OrderStatus.SHIPPED) {
            throw new CannotCancelShippedOrderException();
        }
    }
}

// Application Layer
class CancelOrderUseCase {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;  // Port

    @Transactional
    public Result execute(CancelOrderCommand command) {
        // 1. 조회
        Order order = orderRepository.findById(command.getOrderId());

        // 2. 도메인 로직 실행
        order.cancel();

        // 3. 환불 처리
        paymentService.refund(order.getPaymentId());

        // 4. 저장
        orderRepository.save(order);

        return Result.success();
    }
}
```

### 예시 2: 코드 리뷰

**코드**:
```java
class OrderService {
    public void processOrder(OrderRequest request) {
        // DB 저장
        Order order = new Order();
        order.setUserId(request.getUserId());
        orderRepository.save(order);

        // 결제 처리
        PaymentResponse payment = stripeClient.charge(
            request.getCardNumber(),
            order.getTotalAmount()
        );

        // 이메일 발송
        emailClient.send(
            request.getUserEmail(),
            "주문 완료",
            "주문이 완료되었습니다"
        );
    }
}
```

**문제점 자동 식별**:

```
1. 레이어 혼재
   ❌ 비즈니스 규칙(주문 생성) + 인프라(DB, API, Email)가 섞임

2. 의존성 방향 위반
   ❌ OrderService가 StripeClient 구체 클래스에 의존

3. 단일 책임 위반
   ❌ 주문 처리, 결제, 알림을 모두 담당

4. 높은 결합도
   ❌ 3개의 구체 클래스에 의존
```

**개선 제안**:

```java
// Domain Layer
class Order {
    public static Order create(UserId userId, List<OrderItem> items) {
        Order order = new Order();
        order.id = OrderId.generate();
        order.userId = userId;
        order.items = items;
        order.status = OrderStatus.PENDING;
        order.calculateTotal();
        return order;
    }

    private void calculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// Application Layer
class ProcessOrderUseCase {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;      // Port
    private final NotificationService notificationService;  // Port

    @Transactional
    public Result execute(ProcessOrderCommand command) {
        // 1. 주문 생성
        Order order = Order.create(
            command.getUserId(),
            command.getItems()
        );

        // 2. 결제 처리
        PaymentResult payment = paymentGateway.charge(
            command.getPaymentMethod(),
            order.getTotalAmount()
        );

        // 3. 저장
        orderRepository.save(order);

        // 4. 알림
        notificationService.notifyOrderPlaced(order.getId());

        return Result.success(order.getId());
    }
}

// Infrastructure Layer
class StripePaymentGateway implements PaymentGateway {
    private final StripeClient client;

    @Override
    public PaymentResult charge(PaymentMethod method, Money amount) {
        // Stripe API 호출
    }
}
```

---

## 체크리스트

### 코드 작성 전

- [ ] 이 코드의 변경 이유는 명확한가?
- [ ] 적절한 레이어를 선택했는가?
- [ ] 의존성 방향이 올바른가?
- [ ] 책임이 명확하게 분배되었는가?

### 코드 작성 후

- [ ] 도메인 레이어에 기술 의존성이 없는가?
- [ ] 각 클래스는 단일 책임을 가지는가?
- [ ] 결합도가 낮은가? (인터페이스 사용)
- [ ] 응집도가 높은가? (관련된 것끼리 묶임)
- [ ] 순환 의존성이 없는가?

### 리팩토링 신호

- [ ] 한 클래스가 여러 이유로 변경되는가? → SRP 위반
- [ ] 구체 클래스에 의존하는가? → DIP 위반
- [ ] 도메인이 프레임워크에 의존하는가? → 레이어 위반
- [ ] 비즈니스 로직이 Use Case에 있는가? → 책임 오배치
