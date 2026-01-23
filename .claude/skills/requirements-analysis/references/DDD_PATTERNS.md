# DDD 패턴 참조 가이드

## 전략적 설계 (Strategic Design)

### 바운디드 컨텍스트 (Bounded Context)

**목적**: 도메인 모델의 명확한 경계 설정

**식별 방법**:
1. 팀 구조 기준
2. 비즈니스 기능 기준
3. 유비쿼터스 언어 일관성
4. 변경 빈도와 이유

**패턴**:
```
컨텍스트 관계 유형:

1. Shared Kernel (공유 커널)
   - 두 팀이 코드와 모델 공유
   - 강한 결합, 신중히 사용

2. Customer-Supplier (고객-공급자)
   - Downstream이 Upstream에 의존
   - 명확한 인터페이스 계약

3. Conformist (순응자)
   - Downstream이 Upstream 모델 따름
   - Upstream 변경 불가능할 때

4. Anti-Corruption Layer (부패 방지 계층)
   - Downstream이 격리 계층 구현
   - 레거시 통합 시 유용

5. Open Host Service
   - 프로토콜로 서비스 제공
   - REST API, gRPC 등

6. Published Language
   - 잘 정의된 공통 언어
   - JSON Schema, Protocol Buffers

7. Separate Ways (독립 노선)
   - 통합 없이 독립 운영
   - 중복 허용
```

### 컨텍스트 맵 (Context Map)

**표현 방법**:
```
[Sales Context] --ACL--> [Legacy Billing System]
       |
       | (Events)
       v
[Shipping Context] <--OHS-- [Inventory Context]
```

**범례**:
- ACL: Anti-Corruption Layer
- OHS: Open Host Service
- U/D: Upstream/Downstream
- SK: Shared Kernel

---

## 전술적 설계 (Tactical Design)

### Entity 패턴

**특징**:
- 고유 식별자 보유
- 생명주기 존재
- 가변 상태

**안티패턴**:
```java
// ❌ Anemic Model
class Order {
    private Long id;
    private String status;

    // Getter/Setter만 존재
    public void setStatus(String status) {
        this.status = status;
    }
}

// ✅ Rich Model
class Order {
    private final OrderId id;
    private OrderStatus status;

    public void place() {
        validateCanPlace();
        this.status = OrderStatus.PLACED;
        registerEvent(new OrderPlacedEvent(this.id));
    }

    private void validateCanPlace() {
        if (this.status != null) {
            throw new OrderAlreadyPlacedException();
        }
    }
}
```

### Value Object 패턴

**특징**:
- 속성으로 식별
- 불변
- 교체 가능
- equals/hashCode 구현 필수

**적용 시나리오**:
```
측정값: Money, Temperature, Weight
주소: Address, Location
식별자: Email, PhoneNumber, ISBN
범위: DateRange, AgeRange
복합 속성: PersonName (firstName, lastName)
```

**구현 원칙**:
```java
// 1. 불변성
private final BigDecimal amount;

// 2. Self-Validation
private Money(BigDecimal amount) {
    if (amount == null) {
        throw new IllegalArgumentException("Amount required");
    }
    this.amount = amount;
}

// 3. 정적 팩토리 메서드
public static Money of(BigDecimal amount) {
    return new Money(amount);
}

// 4. 비즈니스 메서드 (새 인스턴스 반환)
public Money add(Money other) {
    return new Money(this.amount.add(other.amount));
}

// 5. Side-Effect-Free Function
public boolean isGreaterThan(Money other) {
    return this.amount.compareTo(other.amount) > 0;
}
```

### Aggregate 패턴

**설계 원칙**:

1. **작게 설계하라**
```
✅ 좋은 예: 하나의 엔티티만
Order (Root)

❌ 나쁜 예: 너무 많은 엔티티
Order (Root)
  └─ OrderItem
       └─ Product
            └─ Category
                 └─ Manufacturer
```

2. **ID로 참조하라**
```java
// ✅ ID 참조
class Order {
    private OrderId id;
    private UserId userId;  // ID만
}

// ❌ 객체 참조
class Order {
    private OrderId id;
    private User user;  // 객체 전체
}
```

3. **트랜잭션 경계**
```
하나의 트랜잭션 = 하나의 Aggregate

✅ Order 저장 시 OrderItem도 함께 저장
❌ Order와 User를 하나의 트랜잭션에서 수정
```

4. **외부 참조 금지**
```java
// ❌ 외부에서 내부 엔티티 직접 수정
Order order = orderRepository.findById(orderId);
OrderItem item = order.getItems().get(0);
item.changeQuantity(5);  // 불변식 깨질 수 있음

// ✅ Aggregate Root를 통한 수정
order.changeItemQuantity(itemId, 5);
```

### Domain Service 패턴

**사용 시점**:
```
✅ 도메인 서비스 사용:
- 여러 Aggregate 조합 연산
- 엔티티나 VO에 속하지 않는 비즈니스 로직
- Stateless 연산

❌ 도메인 서비스 불필요:
- 단일 엔티티 연산 → Entity 메서드
- 외부 시스템 연동 → Application Service
- 유스케이스 조율 → Use Case
```

**구현 패턴**:
```java
// Domain Service
class TransferService {
    public void transfer(Account from, Account to, Money amount) {
        // 1. 검증
        from.validateCanWithdraw(amount);
        to.validateCanDeposit(amount);

        // 2. 도메인 로직
        from.withdraw(amount);
        to.deposit(amount);

        // 3. 이벤트 발행
        publishEvent(new MoneyTransferredEvent(
            from.getId(), to.getId(), amount
        ));
    }
}
```

### Domain Event 패턴

**명명 규칙**:
```
✅ 과거형 (무슨 일이 일어났는가)
OrderPlaced
PaymentCompleted
InventoryDecreased

❌ 명령형
PlaceOrder
CompletePayment
DecreaseInventory
```

**이벤트 설계**:
```java
// 1. 불변 객체
public class OrderPlacedEvent {
    private final String eventId;
    private final LocalDateTime occurredAt;
    private final OrderId orderId;
    private final UserId userId;
    private final Money totalAmount;

    // final 필드, Getter만, Setter 없음
}

// 2. 최소 정보만 포함
// ✅ ID와 핵심 정보만
private final OrderId orderId;
private final Money totalAmount;

// ❌ 전체 엔티티 포함 금지
private final Order order;  // 지양

// 3. 비즈니스 의미 명확
class OrderPlacedEvent {
    // 명확한 비즈니스 이벤트
}

// ❌ 기술적 이벤트
class OrderStatusChangedEvent {
    private String oldStatus;
    private String newStatus;
}
```

### Repository 패턴

**인터페이스 위치**: Domain Layer
**구현 위치**: Infrastructure Layer

**설계 원칙**:
```java
// 1. Aggregate 단위 저장/조회
interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByUserId(UserId userId);
}

// 2. 컬렉션처럼 동작
// ✅ 도메인 중심 메서드명
findByUserId(UserId userId)
findActiveOrders()

// ❌ SQL 같은 메서드명
selectOrdersByUserIdAndStatusEquals(...)

// 3. 도메인 타입 사용
// ✅ Value Object 사용
Order findById(OrderId id);

// ❌ 원시 타입 사용
Order findById(String id);
```

### Factory 패턴

**사용 시점**:
- 복잡한 생성 로직
- 여러 객체 조합 생성
- 생성 규칙 캡슐화

**구현**:
```java
// 1. 정적 팩토리 메서드 (단순한 경우)
class Order {
    public static Order create(UserId userId) {
        return new Order(
            OrderId.generate(),
            userId,
            OrderStatus.DRAFT,
            LocalDateTime.now()
        );
    }
}

// 2. Factory 클래스 (복잡한 경우)
class OrderFactory {
    private final ProductRepository productRepository;

    public Order createFrom(OrderRequest request) {
        User user = validateUser(request.getUserId());
        List<Product> products = loadProducts(request.getItems());

        Order order = Order.create(user.getId());

        for (OrderItemRequest item : request.getItems()) {
            Product product = findProduct(products, item.getProductId());
            order.addItem(product, item.getQuantity());
        }

        return order;
    }
}
```

### Specification 패턴

**목적**: 비즈니스 규칙을 재사용 가능한 객체로 캡슐화

**구현**:
```java
// 1. 기본 인터페이스
interface Specification<T> {
    boolean isSatisfiedBy(T candidate);

    default Specification<T> and(Specification<T> other) {
        return candidate ->
            this.isSatisfiedBy(candidate) &&
            other.isSatisfiedBy(candidate);
    }

    default Specification<T> or(Specification<T> other) {
        return candidate ->
            this.isSatisfiedBy(candidate) ||
            other.isSatisfiedBy(candidate);
    }
}

// 2. 구체적 명세
class OverdueOrderSpecification implements Specification<Order> {
    private static final int OVERDUE_DAYS = 7;

    @Override
    public boolean isSatisfiedBy(Order order) {
        LocalDateTime deadline = order.getPlacedAt()
            .plusDays(OVERDUE_DAYS);
        return LocalDateTime.now().isAfter(deadline);
    }
}

class LargeOrderSpecification implements Specification<Order> {
    private static final Money THRESHOLD = Money.won(100_000);

    @Override
    public boolean isSatisfiedBy(Order order) {
        return order.getTotalAmount().isGreaterThan(THRESHOLD);
    }
}

// 3. 조합 사용
Specification<Order> spec =
    new OverdueOrderSpecification()
        .and(new LargeOrderSpecification());

List<Order> orders = orderRepository.findAll();
List<Order> criticalOrders = orders.stream()
    .filter(spec::isSatisfiedBy)
    .toList();
```

---

## 레이어 패턴

### Layered Architecture

```
[Presentation Layer]
       ↓ (의존)
[Application Layer]
       ↓ (의존)
[Domain Layer]
       ↓ (의존)
[Infrastructure Layer]
```

**의존성 규칙**:
- 상위 레이어는 하위 레이어 의존 가능
- 하위 레이어는 상위 레이어 의존 불가
- Domain Layer는 Infrastructure 의존 불가 (Dependency Inversion)

### Hexagonal Architecture (Ports and Adapters)

```
      [Adapter]
          ↓
      [Port] (Interface)
          ↓
    [Application Core]
          ↓
      [Port] (Interface)
          ↓
      [Adapter]
```

**Port 설계**:
```java
// Driving Port (API)
public interface CreateOrderUseCase {
    OrderResult execute(CreateOrderCommand command);
}

// Driven Port (SPI)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface PaymentGateway {
    PaymentResult process(Payment payment);
}
```

---

## 안티패턴

### 1. Anemic Domain Model
```java
// ❌ 비즈니스 로직이 서비스에만
class Order {
    private String status;
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
}

class OrderService {
    public void placeOrder(Order order) {
        if (order.getStatus() != null) {
            throw new Exception();
        }
        order.setStatus("PLACED");
    }
}
```

### 2. God Object
```java
// ❌ 하나의 클래스가 너무 많은 책임
class Order {
    // 주문 관리
    // 결제 처리
    // 배송 관리
    // 재고 관리
    // 쿠폰 적용
    // 포인트 적립
    // 알림 발송
}
```

### 3. Smart UI
```java
// ❌ UI에 비즈니스 로직
@Controller
class OrderController {
    public String placeOrder(OrderRequest request) {
        Order order = new Order();
        order.setUserId(request.getUserId());

        // 비즈니스 로직이 컨트롤러에
        if (order.getTotalAmount() > 50000) {
            order.setShippingFee(0);
        } else {
            order.setShippingFee(3000);
        }

        orderRepository.save(order);
    }
}
```

### 4. Feature Envy
```java
// ❌ 다른 객체의 데이터를 과도하게 사용
class OrderService {
    public Money calculateTotal(Order order) {
        Money total = Money.ZERO;
        for (OrderItem item : order.getItems()) {
            Money price = item.getProduct().getPrice();
            int quantity = item.getQuantity();
            total = total.add(price.multiply(quantity));
        }
        return total;
    }
}

// ✅ Tell, Don't Ask
class Order {
    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

---

## 실전 패턴 조합

### 주문 처리 시나리오

```java
// 1. Application Service (Use Case)
@UseCase
class PlaceOrderUseCase {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final PriceCalculator priceCalculator;
    private final EventPublisher eventPublisher;

    @Transactional
    public OrderResult execute(PlaceOrderCommand command) {
        // 1. 사용자 조회 및 검증
        User user = loadUser(command.getUserId());

        // 2. 상품 조회
        List<Product> products = loadProducts(command.getItems());

        // 3. 주문 생성 (Factory)
        Order order = Order.create(user.getId());

        // 4. 주문 항목 추가 (Aggregate)
        for (OrderItemCommand item : command.getItems()) {
            Product product = findProduct(products, item.getProductId());
            order.addItem(product, item.getQuantity());
        }

        // 5. 가격 계산 (Domain Service)
        Money finalPrice = priceCalculator.calculateFinalPrice(
            order, user.getGrade(), command.getCoupons()
        );
        order.updateTotalAmount(finalPrice);

        // 6. 주문 확정 (Entity)
        order.place();

        // 7. 저장 (Repository)
        orderRepository.save(order);

        // 8. 이벤트 발행 (Domain Event)
        order.getDomainEvents()
            .forEach(eventPublisher::publish);

        return OrderResult.from(order);
    }
}

// 2. Domain Model
class Order {  // Aggregate Root
    private final OrderId id;
    private OrderStatus status;
    private final List<OrderItem> items = new ArrayList<>();

    public void place() {
        validateCanPlace();
        this.status = OrderStatus.PLACED;
        registerEvent(new OrderPlacedEvent(this.id));
    }
}

// 3. Domain Service
class PriceCalculator {
    public Money calculateFinalPrice(
            Order order,
            MemberGrade grade,
            List<Coupon> coupons) {
        Money itemTotal = order.calculateItemTotal();
        Money afterDiscount = applyDiscounts(itemTotal, grade, coupons);
        Money shippingFee = calculateShippingFee(order);
        return afterDiscount.add(shippingFee);
    }
}
```
