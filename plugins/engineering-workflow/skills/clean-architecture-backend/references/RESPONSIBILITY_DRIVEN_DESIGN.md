# 책임 주도 설계 (Responsibility-Driven Design)

## 목적

객체 지향 설계의 핵심인 책임과 협력을 중심으로 시스템을 설계하는 방법론.

---

## 핵심 개념

### 책임 (Responsibility)

**정의**: 객체가 알아야 하는 정보(knowing)와 수행해야 하는 행동(doing)

**두 가지 유형**:

```
1. Knowing Responsibility (아는 것에 대한 책임)
   - 자신의 데이터에 대해 알기
   - 관련된 객체에 대해 알기
   - 계산할 수 있는 것에 대해 알기

2. Doing Responsibility (하는 것에 대한 책임)
   - 객체 생성이나 계산 수행
   - 다른 객체의 행동 시작 및 제어
   - 다른 객체의 활동 조율
```

**예시**:

```java
class Order {
    // Knowing: 주문 데이터를 알고 있음
    private OrderId id;
    private List<OrderItem> items;
    private Money totalAmount;

    // Doing: 총액 계산을 수행
    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    // Doing: 주문 확정 행동 수행
    public void place() {
        validateCanPlace();
        this.status = OrderStatus.PLACED;
        this.placedAt = LocalDateTime.now();
    }
}
```

### 협력 (Collaboration)

**정의**: 객체들이 메시지를 주고받으며 공동의 목표를 달성하는 과정

```
협력 구조:

Client (요청자)
   │
   │ 메시지
   ↓
Server (응답자)
   │
   │ 메시지
   ↓
Collaborator (협력자)
```

**예시**:

```java
// 협력 시나리오: 주문 생성
class PlaceOrderUseCase {  // Client
    private final OrderFactory factory;
    private final OrderRepository repository;
    private final PriceCalculator calculator;  // Collaborators

    public Result execute(Command command) {
        // 1. Order에게 생성 요청
        Order order = factory.create(command);

        // 2. PriceCalculator에게 가격 계산 요청
        Money finalPrice = calculator.calculate(order);
        order.updateTotal(finalPrice);

        // 3. OrderRepository에게 저장 요청
        repository.save(order);

        return Result.success();
    }
}
```

### 역할 (Role)

**정의**: 관련된 책임의 집합, 교체 가능한 객체들의 추상화

```java
// 역할 정의
interface PaymentGateway {  // 역할
    PaymentResult process(Payment payment);
}

// 역할을 수행하는 다양한 객체들
class StripeGateway implements PaymentGateway { }
class PaypalGateway implements PaymentGateway { }
class TossGateway implements PaymentGateway { }

// 클라이언트는 역할에만 의존
class ProcessPaymentUseCase {
    private final PaymentGateway gateway;  // 역할 의존

    public Result execute(Command command) {
        return gateway.process(command.toPayment());
    }
}
```

---

## 설계 원칙

### 1. Information Expert (정보 전문가)

**원칙**: 정보를 가진 객체가 책임을 진다

```java
// ✅ 정보 전문가 패턴
class Order {
    private List<OrderItem> items;  // 정보를 가짐

    public Money calculateTotal() {  // 정보를 사용하는 책임
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// ❌ 정보 전문가 위반
class OrderService {
    public Money calculateTotal(Order order) {
        Money total = Money.ZERO;
        for (OrderItem item : order.getItems()) {  // Order의 정보를 꺼내서 처리
            total = total.add(item.getPrice().multiply(item.getQuantity()));
        }
        return total;
    }
}
```

**적용 방법**:
1. 필요한 정보가 어디에 있는지 파악
2. 정보를 가진 객체에게 책임 할당
3. 정보를 요청하지 말고, 작업을 요청

### 2. Creator (생성자)

**원칙**: 객체 생성 책임은 다음 조건을 만족하는 객체에게

- 생성될 객체를 포함하거나 관리한다
- 생성될 객체의 초기화 정보를 가진다
- 생성될 객체를 가장 많이 사용한다

```java
// ✅ Creator 패턴
class Order {  // OrderItem을 포함하고 관리
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(Product product, int quantity) {
        // Order가 OrderItem 생성 책임
        OrderItem item = OrderItem.create(
            this.id,
            product,
            quantity
        );
        items.add(item);
    }
}

// ❌ Creator 위반
class OrderService {
    public void addItemToOrder(Order order, Product product, int quantity) {
        // OrderService가 OrderItem 생성 (부적절)
        OrderItem item = new OrderItem(order.getId(), product, quantity);
        order.addItem(item);
    }
}
```

### 3. Controller (제어자)

**원칙**: 시스템 이벤트를 처리할 책임은 Use Case 컨트롤러에게

```java
// ✅ Controller 패턴
class PlaceOrderUseCase {  // Use Case Controller
    public Result execute(PlaceOrderCommand command) {
        // 1. 시스템 이벤트 수신
        // 2. 도메인 객체들에게 작업 위임
        Order order = orderFactory.create(command);
        order.place();
        orderRepository.save(order);

        // 3. 결과 반환
        return Result.success();
    }
}

// ❌ Controller 없이 직접 호출
class OrderController {
    public Response placeOrder(Request request) {
        // 도메인 로직이 Controller에 노출
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setItems(request.getItems());
        orderRepository.save(order);
    }
}
```

### 4. Low Coupling (낮은 결합도)

**원칙**: 객체 간 의존성을 최소화한다

```java
// ✅ Low Coupling - 인터페이스 의존
class PlaceOrderUseCase {
    private final OrderRepository repository;      // 인터페이스
    private final PaymentGateway paymentGateway;   // 인터페이스
    private final NotificationService notifier;     // 인터페이스

    // 의존성 3개, 모두 인터페이스
}

// ❌ High Coupling - 구체 클래스 의존
class PlaceOrderUseCase {
    private final OrderJpaRepository repository;
    private final StripePaymentClient stripeClient;
    private final SendGridEmailClient emailClient;
    private final SlackNotificationClient slackClient;
    private final SmsService smsService;

    // 의존성 5개, 모두 구체 클래스
}
```

**결합도 측정**:
```
낮음 ←────────────────────────────────→ 높음
인터페이스    추상 클래스    구체 클래스    구체 클래스 + 구현 세부사항
```

### 5. High Cohesion (높은 응집도)

**원칙**: 관련된 책임들을 한 곳에 모은다

```java
// ✅ High Cohesion - 주문 관련 책임만
class Order {
    public void place() { }           // 주문 확정
    public void cancel() { }          // 주문 취소
    public Money calculateTotal() { } // 총액 계산
    public void addItem(OrderItem item) { }  // 상품 추가
}

// ❌ Low Cohesion - 관련 없는 책임 혼재
class Order {
    // 주문 관련
    public void place() { }

    // 결제 관련 (Order의 책임 아님)
    public void processPayment() { }
    public void refundPayment() { }

    // 배송 관련 (Order의 책임 아님)
    public void shipOrder() { }
    public void trackShipment() { }

    // 이메일 관련 (Order의 책임 아님)
    public void sendConfirmationEmail() { }
}
```

### 6. Polymorphism (다형성)

**원칙**: 타입에 따라 달라지는 행동은 다형성으로 처리

```java
// ✅ Polymorphism
interface DiscountPolicy {
    Money calculate(Money amount);
}

class PercentageDiscount implements DiscountPolicy {
    public Money calculate(Money amount) {
        return amount.multiply(0.9);  // 10% 할인
    }
}

class FixedDiscount implements DiscountPolicy {
    public Money calculate(Money amount) {
        return amount.subtract(Money.won(1000));  // 1000원 할인
    }
}

class PriceCalculator {
    private final DiscountPolicy policy;  // 다형성

    public Money calculate(Order order) {
        Money total = order.calculateTotal();
        return policy.calculate(total);  // 구체적 타입에 무관
    }
}

// ❌ Polymorphism 없이 분기문
class PriceCalculator {
    public Money calculate(Order order, String discountType) {
        Money total = order.calculateTotal();

        if (discountType.equals("PERCENTAGE")) {
            return total.multiply(0.9);
        } else if (discountType.equals("FIXED")) {
            return total.subtract(Money.won(1000));
        } else {
            return total;
        }
    }
}
```

### 7. Pure Fabrication (순수 가공)

**원칙**: 도메인 개념이 아니어도 설계 품질을 위해 인위적 클래스 생성 가능

```java
// ✅ Pure Fabrication
// OrderRepository는 실제 도메인 개념은 아니지만
// 책임 분리와 테스트 가능성을 위해 도입
interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
}

// PriceCalculator도 순수 가공 클래스
// 복잡한 가격 계산 로직을 분리하기 위해 도입
class PriceCalculator {
    public Money calculate(Order order, List<Discount> discounts) {
        Money total = order.calculateItemTotal();
        for (Discount discount : discounts) {
            total = discount.apply(total);
        }
        return total;
    }
}
```

### 8. Indirection (간접 참조)

**원칙**: 직접 결합을 피하기 위해 중간 객체를 도입

```java
// ✅ Indirection - Port를 통한 간접 참조
// Application Layer
interface PaymentGateway {  // 중간 객체
    PaymentResult process(Payment payment);
}

class PlaceOrderUseCase {
    private final PaymentGateway gateway;  // 간접 참조

    public Result execute(Command command) {
        gateway.process(payment);  // 구체 클래스 모름
    }
}

// Infrastructure Layer
class StripePaymentGateway implements PaymentGateway {
    // 구체적 구현
}

// ❌ 직접 결합
class PlaceOrderUseCase {
    private final StripeApiClient stripeClient;  // 직접 참조

    public Result execute(Command command) {
        stripeClient.charge(...);  // Stripe에 직접 의존
    }
}
```

### 9. Protected Variations (변경 보호)

**원칙**: 변경 가능성이 있는 지점을 인터페이스로 보호

```java
// ✅ Protected Variations
// 외부 시스템이 변경될 수 있음을 예측하고 인터페이스로 보호
interface NotificationService {  // 안정적 인터페이스
    void notify(UserId userId, String message);
}

class EmailNotificationService implements NotificationService { }
class SmsNotificationService implements NotificationService { }
class SlackNotificationService implements NotificationService { }

// 클라이언트는 변경으로부터 보호됨
class PlaceOrderUseCase {
    private final NotificationService notifier;  // 안정적

    public Result execute(Command command) {
        notifier.notify(command.getUserId(), "주문 완료");
        // 알림 방식이 바뀌어도 Use Case는 변경 불필요
    }
}
```

---

## 책임 할당 프로세스

### 1. 도메인 모델 작성

**시작점**: 요구사항에서 명사 추출

```
요구사항:
"사용자가 상품을 장바구니에 담고, 주문을 생성하면
시스템은 결제를 처리하고 주문 확인 이메일을 발송한다"

추출된 개념:
- 사용자 (User)
- 상품 (Product)
- 장바구니 (Cart)
- 주문 (Order)
- 결제 (Payment)
```

### 2. 책임 식별

**방법**: 동사에서 책임을 찾는다

```
"상품을 장바구니에 담는다"
→ Cart의 책임: addProduct()

"주문을 생성한다"
→ Order의 책임: create()

"결제를 처리한다"
→ Payment의 책임: process()

"이메일을 발송한다"
→ NotificationService의 책임: sendEmail()
```

### 3. 협력 설계

**방법**: 메시지를 주고받는 흐름 설계

```
시퀀스:

PlaceOrderUseCase
    ↓ create(command)
  Order
    ↓ validate()
  OrderValidator
    ↓ process(payment)
  PaymentGateway
    ↓ save(order)
  OrderRepository
    ↓ notify(user, message)
  NotificationService
```

### 4. 책임 재배치

**방법**: GRASP 원칙으로 검증하고 조정

```java
// 초기 설계
class PlaceOrderUseCase {
    public Result execute(Command command) {
        // 검증 로직 (너무 많은 책임)
        if (command.getItems().isEmpty()) {
            throw new EmptyOrderException();
        }
        if (command.getTotalAmount() < 0) {
            throw new InvalidAmountException();
        }

        Order order = new Order();
        // 주문 생성 로직도 Use Case에...
    }
}

// 개선: 책임 재배치
class Order {
    public static Order create(CreateOrderCommand command) {
        // Creator: Order가 자신을 생성
        validateCommand(command);
        return new Order(command);
    }

    private static void validateCommand(CreateOrderCommand command) {
        // Information Expert: Order가 자신의 규칙 검증
        if (command.getItems().isEmpty()) {
            throw new EmptyOrderException();
        }
    }
}

class PlaceOrderUseCase {
    public Result execute(Command command) {
        // Controller: 흐름만 제어
        Order order = Order.create(command);
        orderRepository.save(order);
        return Result.success();
    }
}
```

---

## 실전 패턴

### Pattern 1: 메서드 추출로 책임 명확화

```java
// Before: 책임이 불명확
class Order {
    public void place() {
        if (items.isEmpty() || totalAmount.isLessThan(Money.ZERO)) {
            throw new InvalidOrderException();
        }
        this.status = OrderStatus.PLACED;
        this.placedAt = LocalDateTime.now();
    }
}

// After: 책임이 명확
class Order {
    public void place() {
        validateForPlacement();  // 검증 책임 분리
        markAsPlaced();          // 상태 변경 책임 분리
    }

    private void validateForPlacement() {
        validateNotEmpty();
        validatePositiveAmount();
    }

    private void validateNotEmpty() {
        if (items.isEmpty()) {
            throw new EmptyOrderException();
        }
    }

    private void validatePositiveAmount() {
        if (totalAmount.isLessThan(Money.ZERO)) {
            throw new InvalidAmountException();
        }
    }

    private void markAsPlaced() {
        this.status = OrderStatus.PLACED;
        this.placedAt = LocalDateTime.now();
    }
}
```

### Pattern 2: 협력자 도입으로 결합도 낮추기

```java
// Before: 높은 결합도
class PlaceOrderUseCase {
    public Result execute(Command command) {
        Order order = new Order();
        // 직접 검증
        if (!isValidUser(command.getUserId())) {
            throw new InvalidUserException();
        }
        // 직접 재고 확인
        for (OrderItem item : command.getItems()) {
            if (!hasEnoughStock(item.getProductId(), item.getQuantity())) {
                throw new InsufficientStockException();
            }
        }
        // ...
    }
}

// After: 협력자 도입
class PlaceOrderUseCase {
    private final UserValidator userValidator;           // 협력자
    private final InventoryChecker inventoryChecker;     // 협력자

    public Result execute(Command command) {
        userValidator.validate(command.getUserId());
        inventoryChecker.checkAvailability(command.getItems());

        Order order = Order.create(command);
        return Result.success();
    }
}
```

### Pattern 3: 역할 도입으로 확장성 확보

```java
// Before: 구체 클래스 의존
class PlaceOrderUseCase {
    private final StripePaymentClient stripeClient;

    public Result execute(Command command) {
        stripeClient.charge(...);  // Stripe에만 종속
    }
}

// After: 역할 도입
interface PaymentGateway {  // 역할
    PaymentResult process(Payment payment);
}

class StripeGateway implements PaymentGateway { }
class PaypalGateway implements PaymentGateway { }

class PlaceOrderUseCase {
    private final PaymentGateway gateway;  // 역할 의존

    public Result execute(Command command) {
        gateway.process(...);  // 어떤 구현체든 상관없음
    }
}
```

---

## 안티패턴

### Anti-pattern 1: Feature Envy

**문제**: 다른 객체의 데이터를 과도하게 사용

```java
// ❌ Feature Envy
class OrderService {
    public Money calculateDiscount(Order order, User user) {
        // Order와 User의 데이터를 꺼내서 계산
        if (user.getGrade() == UserGrade.VIP) {
            return order.getTotalAmount().multiply(0.2);
        } else if (user.getGrade() == UserGrade.GOLD) {
            return order.getTotalAmount().multiply(0.1);
        }
        return Money.ZERO;
    }
}

// ✅ 개선: 데이터를 가진 객체가 책임
class User {
    public Money calculateDiscount(Money amount) {
        return switch (grade) {
            case VIP -> amount.multiply(0.2);
            case GOLD -> amount.multiply(0.1);
            default -> Money.ZERO;
        };
    }
}

class OrderService {
    public Money calculateDiscount(Order order, User user) {
        return user.calculateDiscount(order.getTotalAmount());
    }
}
```

### Anti-pattern 2: Inappropriate Intimacy

**문제**: 객체 간 지나친 친밀도

```java
// ❌ Inappropriate Intimacy
class Order {
    private List<OrderItem> items;

    public List<OrderItem> getItems() {
        return items;  // 내부 구조 노출
    }
}

class OrderService {
    public void addDiscount(Order order, Discount discount) {
        // Order 내부를 직접 조작
        for (OrderItem item : order.getItems()) {
            item.applyDiscount(discount);
        }
    }
}

// ✅ 개선: Tell, Don't Ask
class Order {
    public void applyDiscount(Discount discount) {
        items.forEach(item -> item.applyDiscount(discount));
    }
}

class OrderService {
    public void addDiscount(Order order, Discount discount) {
        order.applyDiscount(discount);  // 내부 구조 모름
    }
}
```

### Anti-pattern 3: Lazy Class

**문제**: 거의 일을 하지 않는 클래스

```java
// ❌ Lazy Class
class OrderValidator {
    public boolean validate(Order order) {
        return order.getItems().size() > 0;  // 단순 검증만
    }
}

// ✅ 개선: Order로 병합
class Order {
    public void place() {
        validateNotEmpty();  // 직접 검증
        this.status = OrderStatus.PLACED;
    }

    private void validateNotEmpty() {
        if (items.isEmpty()) {
            throw new EmptyOrderException();
        }
    }
}
```

---

## 체크리스트

### 책임 할당 시

- [ ] 이 책임을 수행하는 데 필요한 정보를 누가 가지고 있는가?
- [ ] 이 객체의 생성 정보를 누가 가지고 있는가?
- [ ] 이 작업은 누가 제어해야 하는가?
- [ ] 객체 간 결합도가 낮은가?
- [ ] 객체 내 응집도가 높은가?

### 협력 설계 시

- [ ] 메시지가 명확한가?
- [ ] 메시지의 수신자가 적절한가?
- [ ] 불필요한 중개자는 없는가?
- [ ] 순환 의존성은 없는가?

### 리팩토링 신호

- [ ] 한 클래스가 다른 클래스의 데이터를 많이 사용하는가? → Feature Envy
- [ ] 한 클래스가 거의 일을 하지 않는가? → Lazy Class
- [ ] 클래스 간 지나치게 밀접한가? → Inappropriate Intimacy
- [ ] 한 클래스가 너무 많은 책임을 가지는가? → God Class
