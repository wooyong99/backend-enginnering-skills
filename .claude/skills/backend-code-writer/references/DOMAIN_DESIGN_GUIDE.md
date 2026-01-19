# 도메인 중심 설계 가이드

> **목적**: 효과적인 도메인 모델 설계 및 구현  
> **대상**: 백엔드 개발자 및 AI 개발 어시스턴트  
> **기반**: Domain-Driven Design (DDD)

---

## 📋 목차

1. [도메인 모델 설계 원칙](#도메인-모델-설계-원칙)
2. [엔티티 설계 가이드](#엔티티-설계-가이드)
3. [값 객체 설계 가이드](#값-객체-설계-가이드)
4. [Aggregate 설계 가이드](#aggregate-설계-가이드)
5. [도메인 서비스 설계 가이드](#도메인-서비스-설계-가이드)
6. [도메인 이벤트 설계 가이드](#도메인-이벤트-설계-가이드)
7. [검증 전략](#검증-전략)
8. [도메인 모델 리팩토링](#도메인-모델-리팩토링)

---

## 도메인 모델 설계 원칙

### 유비쿼터스 언어 (Ubiquitous Language)

**원칙**: 도메인 전문가와 개발자가 동일한 언어를 사용한다.

**적용 방법**:

```
비즈니스 용어:
주문하다 → place()
취소하다 → cancel()
승인하다 → approve()
발급하다 → issue()
만료되다 → expire()

❌ 기술 용어 사용 금지:
updateStatus()
changeState()
modifyData()
```

**도메인 용어 매핑 예시**:

```
업무 프로세스        도메인 메서드
-------------------------------------
회원 가입           User.register()
상품 주문           Order.place()
결제 승인           Payment.approve()
배송 시작           Delivery.ship()
주문 취소           Order.cancel()
쿠폰 발급           Coupon.issue()
포인트 적립         Point.accumulate()
포인트 사용         Point.use()
```

### 비즈니스 불변식 (Business Invariants)

**원칙**: 도메인 규칙은 항상 유지되어야 한다.

**불변식 예시**:

```
Order {
    // 불변식: 주문 금액은 항상 0보다 커야 함
    private Money totalAmount;

    public void calculateTotal() {
        Money itemTotal = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);

        if (itemTotal.isLessThanOrEqual(Money.ZERO)) {
            throw new InvalidOrderAmountException(
                "주문 금액은 0보다 커야 합니다"
            );
        }

        this.totalAmount = itemTotal;
    }
}
```

**불변식 보호 전략**:

```
1. 생성 시점 검증
   - 정적 팩토리 메서드에서 검증
   - 생성자는 protected/private

2. 상태 변경 시점 검증
   - 비즈니스 메서드에서 검증
   - Setter 사용 금지

3. 집합(Aggregate) 단위 검증
   - Aggregate Root가 일관성 보장
   - 외부에서 직접 수정 불가
```

### 도메인 주도 (Domain-First)

**원칙**: 도메인 모델을 먼저 설계하고, 기술적 구현은 나중에 고려한다.

**설계 순서**:

```
1단계: 도메인 모델 설계
   ↓
   비즈니스 규칙 정의
   엔티티와 값 객체 식별
   Aggregate 경계 설정

2단계: Use Case 설계
   ↓
   유스케이스 흐름 정의
   Command/Result 설계
   Port 인터페이스 정의

3단계: 기술 구현
   ↓
   데이터베이스 스키마
   API 엔드포인트
   외부 시스템 연동
```

---

## 엔티티 설계 가이드

### 엔티티 식별 기준

**엔티티 정의**: 고유 식별자를 가지며 생명주기가 있는 객체

**엔티티 판단 기준**:

```
✅ 엔티티인 경우:
- 고유 ID가 필요한가?
- 시간에 따라 상태가 변하는가?
- 추적이 필요한가?
- 다른 객체와 구별되어야 하는가?

예시: User, Order, Product, Payment

❌ 엔티티가 아닌 경우:
- 값으로만 비교되는가?
- 불변인가?
- 교체 가능한가?

예시: Money, Email, Address, PhoneNumber
```

### 엔티티 설계 패턴

**기본 구조**:

```
Entity {
    // 1. 식별자 (필수)
    private final EntityId id;

    // 2. 값 객체 (권장)
    private Email email;
    private Money amount;

    // 3. 원시 타입 (최소화)
    private LocalDateTime createdAt;

    // 4. 열거형
    private Status status;

    // 5. 연관 엔티티 (신중히)
    private List<SubEntity> items;

    // 6. 생성자 (protected/private)
    protected Entity() {}

    // 7. 정적 팩토리 메서드
    public static Entity create(...) {
        validate(...);
        return new Entity(...);
    }

    // 8. 비즈니스 메서드
    public void execute() {
        validateCanExecute();
        doExecute();
        raiseEvent(...);
    }

    // 9. 검증 메서드 (private)
    private void validateCanExecute() {
        // 전제조건 검증
    }

    // 10. 이벤트 발행
    private void raiseEvent(...) {
        // 도메인 이벤트
    }
}
```

### 식별자 설계

**식별자 타입**:

```
1. UUID/ULID (권장)
   장점:
   - 분산 시스템 친화적
   - 충돌 가능성 거의 없음
   - 생성 시점에 할당 가능

   단점:
   - 크기가 큼
   - 순차성 없음

2. 자동 증가 (Auto Increment)
   장점:
   - 작은 크기
   - 순차적
   - 인덱스 효율적

   단점:
   - 중앙 집중식
   - 예측 가능
   - 분산 환경 부적합

3. 비즈니스 식별자
   장점:
   - 의미 있음
   - 사용자 친화적

   단점:
   - 변경 가능성
   - 충돌 가능성
```

**식별자 구현**:

```
// 값 객체로 식별자 래핑
class OrderId {
    private final String value;

    private OrderId(String value) {
        validateFormat(value);
        this.value = value;
    }

    public static OrderId generate() {
        return new OrderId(ULID.random());
    }

    public static OrderId of(String value) {
        return new OrderId(value);
    }

    private void validateFormat(String value) {
        if (!isValidULID(value)) {
            throw new InvalidOrderIdException(value);
        }
    }

    // equals, hashCode 구현
}

// 엔티티에서 사용
class Order {
    private final OrderId id;

    public static Order create(...) {
        return new Order(OrderId.generate(), ...);
    }
}
```

### 상태 전이 관리

**상태 전이 패턴**:

```
Order {
    private OrderStatus status;

    // ✅ 명시적 상태 전이 메서드
    public void place() {
        validateCanPlace();
        this.status = OrderStatus.PLACED;
        this.placedAt = LocalDateTime.now();
        raiseEvent(new OrderPlacedEvent(this.id));
    }

    public void pay() {
        validateCanPay();
        this.status = OrderStatus.PAID;
        this.paidAt = LocalDateTime.now();
        raiseEvent(new OrderPaidEvent(this.id));
    }

    public void cancel() {
        validateCanCancel();
        this.status = OrderStatus.CANCELLED;
        this.cancelledAt = LocalDateTime.now();
        raiseEvent(new OrderCancelledEvent(this.id));
    }

    // 전제조건 검증
    private void validateCanPlace() {
        if (this.status != null) {
            throw new OrderAlreadyPlacedException();
        }
    }

    private void validateCanPay() {
        if (this.status != OrderStatus.PLACED) {
            throw new InvalidOrderStatusException(
                "결제는 주문 완료 상태에서만 가능합니다"
            );
        }
    }

    private void validateCanCancel() {
        if (this.status == OrderStatus.SHIPPED) {
            throw new OrderAlreadyShippedException(
                "배송 시작된 주문은 취소할 수 없습니다"
            );
        }
    }
}
```

**상태 머신 패턴** (복잡한 경우):

```
enum OrderStatus {
    PLACED {
        @Override
        public boolean canTransitionTo(OrderStatus next) {
            return next == PAID || next == CANCELLED;
        }
    },
    PAID {
        @Override
        public boolean canTransitionTo(OrderStatus next) {
            return next == SHIPPED || next == REFUNDED;
        }
    },
    SHIPPED {
        @Override
        public boolean canTransitionTo(OrderStatus next) {
            return next == DELIVERED || next == RETURNED;
        }
    },
    // ...

    public abstract boolean canTransitionTo(OrderStatus next);
}

Order {
    public void transitionTo(OrderStatus nextStatus) {
        if (!this.status.canTransitionTo(nextStatus)) {
            throw new InvalidStatusTransitionException(
                this.status, nextStatus
            );
        }
        this.status = nextStatus;
    }
}
```

### 컬렉션 관리

**컬렉션 캡슐화**:

```
Order {
    private final List<OrderItem> items = new ArrayList<>();

    // ❌ 잘못된 방법
    public List<OrderItem> getItems() {
        return items;  // 외부에서 직접 수정 가능
    }

    // ✅ 올바른 방법 1: 읽기 전용 반환
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }

    // ✅ 올바른 방법 2: 비즈니스 메서드 제공
    public void addItem(Product product, int quantity) {
        validateCanAddItem(product);

        OrderItem item = OrderItem.create(product, quantity);
        items.add(item);

        recalculateTotal();
    }

    public void removeItem(OrderItemId itemId) {
        OrderItem item = findItem(itemId);
        items.remove(item);
        recalculateTotal();
    }

    private OrderItem findItem(OrderItemId itemId) {
        return items.stream()
            .filter(item -> item.getId().equals(itemId))
            .findFirst()
            .orElseThrow(() -> new OrderItemNotFoundException(itemId));
    }
}
```

---

## 값 객체 설계 가이드

### 값 객체 식별 기준

**값 객체 정의**: 속성으로만 식별되는 불변 객체

**값 객체 판단 기준**:

```
✅ 값 객체인 경우:
- 측정, 수량, 설명인가?
- 불변인가?
- 값으로만 비교되는가?
- 교체 가능한가?

예시: Money, Email, PhoneNumber, Address, DateRange

❌ 값 객체가 아닌 경우:
- 고유 식별자가 필요한가?
- 생명주기가 있는가?
- 추적이 필요한가?

예시: User, Order, Product
```

### 값 객체 설계 패턴

**기본 구조**:

```
ValueObject {
    // 1. final 필드 (불변)
    private final Type value;

    // 2. private 생성자
    private ValueObject(Type value) {
        validate(value);
        this.value = value;
    }

    // 3. 정적 팩토리 메서드
    public static ValueObject of(Type value) {
        return new ValueObject(value);
    }

    // 4. 검증
    private void validate(Type value) {
        // 값 검증
    }

    // 5. 비즈니스 메서드 (새 인스턴스 반환)
    public ValueObject operation(ValueObject other) {
        Type result = this.value.operation(other.value);
        return new ValueObject(result);
    }

    // 6. equals, hashCode 구현 (필수)
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ValueObject)) return false;
        ValueObject that = (ValueObject) o;
        return Objects.equals(value, that.value);
    }

    @Override
    public int hashCode() {
        return Objects.hash(value);
    }
}
```

### 대표적인 값 객체 예시

**Money (금액)**:

```
class Money {
    private final BigDecimal amount;
    private final Currency currency;

    private Money(BigDecimal amount, Currency currency) {
        validateAmount(amount);
        validateCurrency(currency);
        this.amount = amount;
        this.currency = currency;
    }

    public static Money of(BigDecimal amount, Currency currency) {
        return new Money(amount, currency);
    }

    public static Money won(long amount) {
        return new Money(
            BigDecimal.valueOf(amount),
            Currency.getInstance("KRW")
        );
    }

    public static Money ZERO = won(0);

    // 비즈니스 메서드
    public Money add(Money other) {
        validateSameCurrency(other);
        return new Money(
            this.amount.add(other.amount),
            this.currency
        );
    }

    public Money subtract(Money other) {
        validateSameCurrency(other);
        return new Money(
            this.amount.subtract(other.amount),
            this.currency
        );
    }

    public Money multiply(BigDecimal multiplier) {
        return new Money(
            this.amount.multiply(multiplier),
            this.currency
        );
    }

    public boolean isGreaterThan(Money other) {
        validateSameCurrency(other);
        return this.amount.compareTo(other.amount) > 0;
    }

    public boolean isLessThan(Money other) {
        validateSameCurrency(other);
        return this.amount.compareTo(other.amount) < 0;
    }

    private void validateSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new DifferentCurrencyException(
                this.currency, other.currency
            );
        }
    }
}
```

**Email**:

```
class Email {
    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("^[A-Za-z0-9+_.-]+@(.+)$");

    private final String value;

    private Email(String value) {
        validate(value);
        this.value = value.toLowerCase();
    }

    public static Email of(String value) {
        return new Email(value);
    }

    private void validate(String value) {
        if (value == null || value.isBlank()) {
            throw new InvalidEmailException("이메일은 필수입니다");
        }

        if (!EMAIL_PATTERN.matcher(value).matches()) {
            throw new InvalidEmailException(
                "올바르지 않은 이메일 형식입니다: " + value
            );
        }
    }

    public String getDomain() {
        return value.substring(value.indexOf('@') + 1);
    }

    public String getLocalPart() {
        return value.substring(0, value.indexOf('@'));
    }
}
```

**DateRange (기간)**:

```
class DateRange {
    private final LocalDate startDate;
    private final LocalDate endDate;

    private DateRange(LocalDate startDate, LocalDate endDate) {
        validate(startDate, endDate);
        this.startDate = startDate;
        this.endDate = endDate;
    }

    public static DateRange of(LocalDate startDate, LocalDate endDate) {
        return new DateRange(startDate, endDate);
    }

    public static DateRange ofDays(LocalDate startDate, int days) {
        return new DateRange(
            startDate,
            startDate.plusDays(days)
        );
    }

    private void validate(LocalDate startDate, LocalDate endDate) {
        if (startDate == null || endDate == null) {
            throw new InvalidDateRangeException("날짜는 필수입니다");
        }

        if (startDate.isAfter(endDate)) {
            throw new InvalidDateRangeException(
                "시작일은 종료일보다 이전이어야 합니다"
            );
        }
    }

    public boolean contains(LocalDate date) {
        return !date.isBefore(startDate) && !date.isAfter(endDate);
    }

    public boolean overlaps(DateRange other) {
        return !this.endDate.isBefore(other.startDate) &&
               !other.endDate.isBefore(this.startDate);
    }

    public long getDays() {
        return ChronoUnit.DAYS.between(startDate, endDate) + 1;
    }
}
```

---

## Aggregate 설계 가이드

### Aggregate 설계 원칙

**Aggregate 정의**: 트랜잭션 일관성 경계를 가진 엔티티 클러스터

**설계 원칙**:

```
1. 작게 설계하라
   - 성능 고려
   - 동시성 충돌 최소화
   - 하나의 엔티티만 포함 가능

2. ID로 참조하라
   - 다른 Aggregate는 ID로만 참조
   - 직접 객체 참조 금지
   - 느슨한 결합 유지

3. 경계 밖에서는 eventual consistency
   - Aggregate 내부: 강한 일관성
   - Aggregate 외부: 최종적 일관성
   - 이벤트로 전파

4. 하나의 트랜잭션은 하나의 Aggregate만
   - 동시성 제어 단순화
   - 성능 향상
```

### Aggregate Root 설계

**Aggregate Root 책임**:

```
AggregateRoot {
    // 1. 일관성 보장
    - 내부 엔티티 캡슐화
    - 불변식 유지
    - 트랜잭션 경계

    // 2. 외부 진입점
    - 모든 작업은 Root를 통해
    - 내부 엔티티 직접 접근 불가

    // 3. 도메인 이벤트 발행
    - 상태 변경 시 이벤트 발행
    - 다른 Aggregate 통지
}
```

**설계 예시**:

```
// Aggregate Root
Order {
    private final OrderId id;
    private OrderStatus status;
    private Money totalAmount;

    // 내부 엔티티 (캡슐화)
    private final List<OrderItem> items = new ArrayList<>();

    // 외부 진입점
    public void addItem(Product product, int quantity) {
        // 불변식 검증
        validateCanAddItem(product);

        // 내부 엔티티 생성
        OrderItem item = OrderItem.create(this.id, product, quantity);
        items.add(item);

        // 일관성 유지
        recalculateTotal();
    }

    public void removeItem(OrderItemId itemId) {
        OrderItem item = findItem(itemId);
        items.remove(item);
        recalculateTotal();
    }

    // 불변식 검증
    private void validateCanAddItem(Product product) {
        if (this.status != OrderStatus.DRAFT) {
            throw new OrderAlreadyPlacedException();
        }

        if (!product.isAvailable()) {
            throw new ProductNotAvailableException();
        }
    }

    // 일관성 유지
    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    // ❌ 내부 엔티티 직접 노출 금지
    // public List<OrderItem> getItems() { ... }

    // ✅ 읽기 전용 메서드 제공
    public int getItemCount() {
        return items.size();
    }

    public boolean hasItem(ProductId productId) {
        return items.stream()
            .anyMatch(item -> item.getProductId().equals(productId));
    }
}

// 내부 엔티티 (외부에서 직접 생성 불가)
OrderItem {
    private final OrderItemId id;
    private final OrderId orderId;  // Aggregate Root 참조
    private final ProductId productId;  // ID로 참조
    private final Money unitPrice;
    private int quantity;

    // package-private 생성자
    static OrderItem create(
            OrderId orderId,
            Product product,
            int quantity) {
        return new OrderItem(
            OrderItemId.generate(),
            orderId,
            product.getId(),
            product.getPrice(),
            quantity
        );
    }

    Money getSubtotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }
}
```

### Aggregate 간 참조

**ID로 참조** (권장):

```
// ✅ ID로 참조
Order {
    private final OrderId id;
    private final UserId userId;  // ID로만 참조

    public Order(OrderId id, UserId userId) {
        this.id = id;
        this.userId = userId;
    }
}

// Use Case에서 필요시 조회
CreateOrderUseCase {
    public Result execute(Command command) {
        User user = userRepository.findById(command.getUserId());
        validateUser(user);

        Order order = Order.create(
            command.getOrderId(),
            user.getId()  // ID만 전달
        );

        orderRepository.save(order);
    }
}
```

**객체로 참조** (지양):

```
// ❌ 객체로 직접 참조 (지양)
Order {
    private final OrderId id;
    private final User user;  // 객체 참조

    // 문제점:
    // 1. Order 조회 시 항상 User도 조회 (N+1)
    // 2. 트랜잭션 경계 모호
    // 3. User 변경 시 Order도 영향
}
```

---

## 도메인 서비스 설계 가이드

### 도메인 서비스 사용 시점

**사용 기준**:

```
✅ 도메인 서비스가 필요한 경우:
- 여러 엔티티를 조합한 로직
- 엔티티에 속하지 않는 계산
- 순수 비즈니스 규칙

❌ 도메인 서비스가 불필요한 경우:
- 단일 엔티티 규칙 → Entity 메서드
- 외부 시스템 접근 필요 → Application Service
- 유스케이스 흐름 제어 → Use Case
```

### 도메인 서비스 vs Application 서비스

| 특성      | 도메인 서비스 | Application 서비스    |
| --------- | ------------- | --------------------- |
| 위치      | Domain Layer  | Application Layer     |
| 의존성    | 도메인 객체만 | Port, Repository 가능 |
| 외부 접근 | 불가          | 가능 (Port 통해)      |
| 트랜잭션  | 없음          | 있음                  |
| 상태      | Stateless     | Stateless             |

### 도메인 서비스 구현

**가격 계산 서비스**:

```
// Domain Service
PriceCalculator {
    /**
     * 주문 최종 가격 계산
     * - 상품 가격 합계
     * - 회원 등급 할인
     * - 쿠폰 할인
     * - 배송비
     */
    public Money calculateFinalPrice(
            Order order,
            MemberGrade memberGrade,
            List<Coupon> coupons) {

        // 1. 상품 가격 합계
        Money itemTotal = order.calculateItemTotal();

        // 2. 회원 등급 할인
        Money afterMemberDiscount =
            applyMemberDiscount(itemTotal, memberGrade);

        // 3. 쿠폰 할인
        Money afterCouponDiscount =
            applyCoupons(afterMemberDiscount, coupons);

        // 4. 배송비 계산
        Money shippingFee = calculateShippingFee(order);

        // 5. 최종 금액
        return afterCouponDiscount.add(shippingFee);
    }

    private Money applyMemberDiscount(
            Money amount,
            MemberGrade grade) {
        BigDecimal rate = switch (grade) {
            case VIP -> BigDecimal.valueOf(0.15);      // 15%
            case GOLD -> BigDecimal.valueOf(0.10);     // 10%
            case SILVER -> BigDecimal.valueOf(0.05);   // 5%
            case BRONZE -> BigDecimal.ZERO;            // 0%
        };

        Money discount = amount.multiply(rate);
        return amount.subtract(discount);
    }

    private Money applyCoupons(
            Money amount,
            List<Coupon> coupons) {
        Money result = amount;

        for (Coupon coupon : coupons) {
            result = coupon.apply(result);
        }

        return result;
    }

    private Money calculateShippingFee(Order order) {
        Money FREE_SHIPPING_THRESHOLD = Money.won(50000);
        Money STANDARD_SHIPPING_FEE = Money.won(3000);

        if (order.getTotalAmount().isGreaterThanOrEqual(
                FREE_SHIPPING_THRESHOLD)) {
            return Money.ZERO;
        }

        return STANDARD_SHIPPING_FEE;
    }
}
```

**정책 서비스**:

```
// Domain Service
RefundPolicy {
    /**
     * 환불 가능 여부 및 환불 금액 계산
     */
    public RefundResult calculate(Order order, LocalDateTime requestedAt) {
        // 1. 환불 가능 검증
        validateRefundable(order, requestedAt);

        // 2. 환불 금액 계산
        Money refundAmount = calculateRefundAmount(order, requestedAt);

        // 3. 수수료 계산
        Money fee = calculateRefundFee(order, requestedAt);

        // 4. 최종 환불 금액
        Money finalAmount = refundAmount.subtract(fee);

        return new RefundResult(true, finalAmount, fee);
    }

    private void validateRefundable(
            Order order,
            LocalDateTime requestedAt) {

        // 배송 완료 후 7일 이내만 환불 가능
        if (order.getDeliveredAt() == null) {
            throw new OrderNotDeliveredException();
        }

        LocalDateTime deadline = order.getDeliveredAt().plusDays(7);
        if (requestedAt.isAfter(deadline)) {
            throw new RefundPeriodExpiredException();
        }
    }

    private Money calculateRefundAmount(
            Order order,
            LocalDateTime requestedAt) {

        // 24시간 이내: 전액 환불
        LocalDateTime fullRefundDeadline =
            order.getDeliveredAt().plusHours(24);

        if (requestedAt.isBefore(fullRefundDeadline)) {
            return order.getTotalAmount();
        }

        // 그 외: 배송비 제외
        return order.getTotalAmount()
            .subtract(order.getShippingFee());
    }

    private Money calculateRefundFee(
            Order order,
            LocalDateTime requestedAt) {

        // 24시간 이내: 수수료 없음
        LocalDateTime freeRefundDeadline =
            order.getDeliveredAt().plusHours(24);

        if (requestedAt.isBefore(freeRefundDeadline)) {
            return Money.ZERO;
        }

        // 그 외: 2,500원 수수료
        return Money.won(2500);
    }
}
```

---

## 도메인 이벤트 설계 가이드

### 도메인 이벤트 원칙

**원칙**:

```
1. 과거형으로 명명
   OrderCreated (O)
   CreateOrder (X)

2. 불변 객체
   발행 후 변경 불가

3. 필요한 정보만 포함
   전체 엔티티 포함 금지
   ID와 핵심 정보만

4. 도메인 용어 사용
   비즈니스 의미 명확히
```

### 이벤트 설계 패턴

**기본 구조**:

```
DomainEvent {
    // 1. 이벤트 메타데이터
    private final String eventId;
    private final LocalDateTime occurredAt;

    // 2. Aggregate 식별자
    private final AggregateId aggregateId;

    // 3. 이벤트 페이로드
    private final EventData data;

    protected DomainEvent(
            AggregateId aggregateId,
            EventData data) {
        this.eventId = UUID.randomUUID().toString();
        this.occurredAt = LocalDateTime.now();
        this.aggregateId = aggregateId;
        this.data = data;
    }
}
```

**구체적 이벤트 예시**:

```
// 주문 생성 이벤트
OrderCreatedEvent extends DomainEvent {
    private final OrderId orderId;
    private final UserId userId;
    private final Money totalAmount;
    private final int itemCount;

    public OrderCreatedEvent(
            OrderId orderId,
            UserId userId,
            Money totalAmount,
            int itemCount) {
        super(orderId);
        this.orderId = orderId;
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.itemCount = itemCount;
    }

    // Getter만 제공 (불변)
}

// 결제 완료 이벤트
PaymentCompletedEvent extends DomainEvent {
    private final PaymentId paymentId;
    private final OrderId orderId;
    private final Money amount;
    private final PaymentMethod method;

    public PaymentCompletedEvent(
            PaymentId paymentId,
            OrderId orderId,
            Money amount,
            PaymentMethod method) {
        super(paymentId);
        this.paymentId = paymentId;
        this.orderId = orderId;
        this.amount = amount;
        this.method = method;
    }
}
```

### 이벤트 발행 시점

**엔티티 내부에서 발행**:

```
Order {
    private List<DomainEvent> domainEvents = new ArrayList<>();

    public void place() {
        validateCanPlace();

        this.status = OrderStatus.PLACED;
        this.placedAt = LocalDateTime.now();

        // 이벤트 등록
        registerEvent(new OrderPlacedEvent(
            this.id,
            this.userId,
            this.totalAmount,
            this.items.size()
        ));
    }

    private void registerEvent(DomainEvent event) {
        domainEvents.add(event);
    }

    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearDomainEvents() {
        domainEvents.clear();
    }
}
```

**Use Case에서 발행**:

```
PlaceOrderUseCase {
    private final OrderRepository orderRepository;
    private final EventPublisher eventPublisher;

    @Transactional
    public OrderResult execute(PlaceOrderCommand command) {
        // 1. 주문 생성
        Order order = Order.create(command);
        order.place();

        // 2. 저장
        orderRepository.save(order);

        // 3. 이벤트 발행
        order.getDomainEvents().forEach(eventPublisher::publish);
        order.clearDomainEvents();

        return OrderResult.from(order);
    }
}
```

---

## 검증 전략

### 검증 레벨

```
1. 값 객체 생성 시
   - 형식 검증
   - 범위 검증
   - 필수값 검증

2. 엔티티 생성 시
   - 비즈니스 규칙 검증
   - 불변식 검증

3. 도메인 메서드 실행 시
   - 전제조건 검증
   - 상태 검증

4. Use Case 실행 시
   - 외부 데이터 검증
   - 권한 검증
```

### 검증 구현 패턴

**값 객체 검증**:

```
Email {
    private Email(String value) {
        validateNotNull(value);
        validateFormat(value);
        this.value = normalize(value);
    }

    private void validateNotNull(String value) {
        if (value == null || value.isBlank()) {
            throw new InvalidEmailException("이메일은 필수입니다");
        }
    }

    private void validateFormat(String value) {
        if (!EMAIL_PATTERN.matcher(value).matches()) {
            throw new InvalidEmailException(
                "올바르지 않은 이메일 형식입니다: " + value
            );
        }
    }

    private String normalize(String value) {
        return value.toLowerCase().trim();
    }
}
```

**엔티티 검증**:

```
Order {
    public static Order create(CreateOrderCommand command) {
        validateCommand(command);

        Order order = new Order();
        order.id = OrderId.generate();
        order.userId = command.getUserId();
        order.status = OrderStatus.DRAFT;

        // 불변식 검증
        order.validateInvariants();

        return order;
    }

    private static void validateCommand(CreateOrderCommand command) {
        if (command.getUserId() == null) {
            throw new InvalidOrderException("사용자 ID는 필수입니다");
        }
    }

    private void validateInvariants() {
        if (this.totalAmount != null &&
            this.totalAmount.isLessThan(Money.ZERO)) {
            throw new InvalidOrderAmountException(
                "주문 금액은 0보다 작을 수 없습니다"
            );
        }
    }
}
```

**Use Case 검증**:

```
CreateOrderUseCase {
    private final OrderValidator validator;

    public OrderResult execute(CreateOrderCommand command) {
        // 복잡한 검증은 Validator에 위임
        validator.validateForCreate(command);

        Order order = Order.create(command);
        orderRepository.save(order);

        return OrderResult.from(order);
    }
}

OrderValidator {
    private final UserRepository userRepository;
    private final ProductRepository productRepository;

    public void validateForCreate(CreateOrderCommand command) {
        // 1. 사용자 존재 검증
        User user = userRepository.findById(command.getUserId())
            .orElseThrow(() -> new UserNotFoundException());

        // 2. 사용자 상태 검증
        if (user.isDeleted()) {
            throw new DeletedUserException();
        }

        // 3. 상품 재고 검증
        for (OrderItemCommand item : command.getItems()) {
            Product product = productRepository.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException());

            if (product.getStock() < item.getQuantity()) {
                throw new InsufficientStockException();
            }
        }
    }
}
```

---

## 도메인 모델 리팩토링

### 리팩토링 신호

**리팩토링이 필요한 징후**:

```
1. Anemic Domain Model
   - Getter/Setter만 존재
   - 비즈니스 로직이 서비스에

2. God Entity
   - 하나의 엔티티가 너무 많은 책임
   - 100줄 이상의 메서드

3. Primitive Obsession
   - 원시 타입 남발
   - String, int, long 등

4. Feature Envy
   - 다른 객체의 데이터를 과도하게 사용
   - Getter 체이닝

5. 중복 코드
   - 같은 로직이 여러 곳에
```

### 리팩토링 패턴

**패턴 1: 값 객체 추출**:

```
// Before
Order {
    private BigDecimal amount;
    private String currency;

    public void applyDiscount(BigDecimal rate) {
        this.amount = this.amount.multiply(
            BigDecimal.ONE.subtract(rate)
        );
    }
}

// After
Order {
    private Money totalAmount;

    public void applyDiscount(DiscountRate rate) {
        this.totalAmount = this.totalAmount.applyDiscount(rate);
    }
}

Money {
    public Money applyDiscount(DiscountRate rate) {
        return this.multiply(BigDecimal.ONE.subtract(rate.getValue()));
    }
}
```

**패턴 2: 도메인 서비스 추출**:

```
// Before
Order {
    public Money calculateFinalPrice(
            List<Coupon> coupons,
            MemberGrade grade) {
        // 복잡한 가격 계산 로직 50줄
    }
}

// After
Order {
    public Money calculateItemTotal() {
        // 상품 합계만 계산
    }
}

PriceCalculator {
    public Money calculateFinalPrice(
            Order order,
            List<Coupon> coupons,
            MemberGrade grade) {
        Money itemTotal = order.calculateItemTotal();
        // 복잡한 계산 로직
    }
}
```

**패턴 3: 정책 객체 추출**:

```
// Before
Order {
    public Money calculateShippingFee() {
        if (this.totalAmount.isGreaterThan(Money.won(50000))) {
            return Money.ZERO;
        }
        if (this.destination.isJejuIsland()) {
            return Money.won(5000);
        }
        return Money.won(3000);
    }
}

// After
ShippingFeePolicy {
    private static final Money FREE_THRESHOLD = Money.won(50000);
    private static final Money STANDARD_FEE = Money.won(3000);
    private static final Money JEJU_FEE = Money.won(5000);

    public Money calculate(Order order) {
        if (order.getTotalAmount().isGreaterThan(FREE_THRESHOLD)) {
            return Money.ZERO;
        }

        if (order.getDestination().isJejuIsland()) {
            return JEJU_FEE;
        }

        return STANDARD_FEE;
    }
}
```

---

이 가이드는 효과적인 도메인 모델 설계와 구현을 위한 실용적인 지침을 제공합니다. 프로젝트의 특성과 팀의 상황에 맞게 조정하여 사용하시기 바랍니다.
