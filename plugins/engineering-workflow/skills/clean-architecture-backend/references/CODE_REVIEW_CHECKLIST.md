# 코드 리뷰 체크리스트

## 목적

클린 아키텍처와 객체 지향 설계 관점에서 코드를 체계적으로 리뷰하기 위한 체크리스트.

---

## 1. 아키텍처 준수

### 1.1 레이어 분리

```
□ Domain Layer가 다른 레이어에 의존하지 않는가?
□ Application Layer가 Infrastructure를 직접 의존하지 않는가?
□ 각 클래스가 적절한 레이어에 배치되었는가?
□ 레이어 간 경계가 명확한가?
```

**검증 방법**:
```java
// ❌ 레이어 위반
package com.shop.domain;
import javax.persistence.*;  // Infrastructure 의존

@Entity
class Order { }

// ✅ 레이어 준수
package com.shop.domain;
// 순수 Java만 사용
class Order { }
```

### 1.2 의존성 방향

```
□ 의존성이 안정된 방향(Domain)으로 향하는가?
□ Port 인터페이스가 올바른 위치에 있는가?
□ DIP가 적절히 적용되었는가?
□ 순환 의존성이 없는가?
```

**검증 방법**:
```java
// ✅ 올바른 의존성 방향
Application ──→ Domain
    ↑
    │ implements
    │
Infrastructure
```

---

## 2. 도메인 모델

### 2.1 엔티티

```
□ 엔티티가 고유 식별자를 가지는가?
□ 비즈니스 메서드가 풍부한가? (Anemic Model 아님)
□ 불변식이 보호되고 있는가?
□ 상태 전이가 명시적인가?
□ Getter/Setter만 있지 않은가?
```

**검증 예시**:
```java
// ❌ Anemic Model
class Order {
    private OrderStatus status;

    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }
}

// ✅ Rich Model
class Order {
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

### 2.2 값 객체

```
□ 값 객체가 불변인가?
□ equals/hashCode가 구현되어 있는가?
□ 원시 타입 집착을 피했는가?
□ 검증 로직이 포함되어 있는가?
□ 비즈니스 의미가 명확한가?
```

**검증 예시**:
```java
// ❌ 원시 타입 사용
class Order {
    private BigDecimal amount;
    private String currency;
}

// ✅ 값 객체 사용
class Order {
    private Money totalAmount;
}

class Money {
    private final BigDecimal amount;
    private final Currency currency;

    // 불변, equals/hashCode 구현, 검증 로직 포함
}
```

### 2.3 Aggregate

```
□ Aggregate 경계가 명확한가?
□ Aggregate Root를 통해서만 접근하는가?
□ 다른 Aggregate를 ID로 참조하는가?
□ 트랜잭션 일관성 경계가 적절한가?
□ Aggregate 크기가 적절한가? (작게 유지)
```

**검증 예시**:
```java
// ✅ 올바른 Aggregate
class Order {  // Aggregate Root
    private OrderId id;
    private UserId userId;  // 다른 Aggregate는 ID로 참조
    private List<OrderItem> items;  // 내부 엔티티

    public void addItem(Product product, int quantity) {
        // Root를 통한 접근
        OrderItem item = OrderItem.create(this.id, product, quantity);
        items.add(item);
        recalculateTotal();
    }

    // ❌ 외부에서 직접 접근 불가
    // public List<OrderItem> getItems() { return items; }
}
```

---

## 3. 책임과 협력

### 3.1 단일 책임 원칙 (SRP)

```
□ 클래스가 하나의 변경 이유만 가지는가?
□ 메서드가 하나의 작업만 수행하는가?
□ 클래스 이름이 책임을 명확히 나타내는가?
□ 클래스 크기가 적절한가? (300줄 이하 권장)
```

**검증 예시**:
```java
// ❌ SRP 위반
class OrderService {
    public void processOrder(Order order) {
        // 검증
        validate(order);
        // 저장
        save(order);
        // 결제
        processPayment(order);
        // 이메일
        sendEmail(order);
        // 재고
        updateInventory(order);
    }
}

// ✅ SRP 준수
class PlaceOrderUseCase {  // 주문 확정만
    public Result execute(Command command) {
        Order order = Order.create(command);
        order.place();
        orderRepository.save(order);
        return Result.success();
    }
}

class ProcessPaymentUseCase {  // 결제만
    // ...
}

class NotifyOrderPlacedUseCase {  // 알림만
    // ...
}
```

### 3.2 정보 전문가 (Information Expert)

```
□ 정보를 가진 객체가 행동의 책임을 지는가?
□ Tell, Don't Ask 원칙을 따르는가?
□ Feature Envy가 없는가?
```

**검증 예시**:
```java
// ❌ Feature Envy
class OrderService {
    public Money calculateTotal(Order order) {
        Money total = Money.ZERO;
        for (OrderItem item : order.getItems()) {
            total = total.add(item.getPrice().multiply(item.getQuantity()));
        }
        return total;
    }
}

// ✅ Information Expert
class Order {
    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

### 3.3 낮은 결합도 (Low Coupling)

```
□ 인터페이스에 의존하는가?
□ 의존성 개수가 적절한가? (5개 이하 권장)
□ 구체 클래스 직접 의존을 피했는가?
□ Law of Demeter를 준수하는가?
```

**검증 예시**:
```java
// ❌ High Coupling
class PlaceOrderUseCase {
    private final OrderJpaRepository repository;
    private final StripeApiClient stripeClient;
    private final SendGridClient emailClient;
    // 모두 구체 클래스 의존
}

// ✅ Low Coupling
class PlaceOrderUseCase {
    private final OrderRepository repository;  // 인터페이스
    private final PaymentGateway gateway;      // 인터페이스
    private final NotificationService notifier; // 인터페이스
}
```

### 3.4 높은 응집도 (High Cohesion)

```
□ 클래스 내 메서드들이 관련되어 있는가?
□ 데이터와 행동이 함께 있는가?
□ 관련 없는 책임이 섞여 있지 않은가?
```

**검증 예시**:
```java
// ❌ Low Cohesion
class Order {
    // 주문 관련
    public void place() { }
    // 결제 관련 (관련 없음)
    public void processPayment() { }
    // 배송 관련 (관련 없음)
    public void shipOrder() { }
}

// ✅ High Cohesion
class Order {
    // 주문 관련 책임만
    public void place() { }
    public void cancel() { }
    public Money calculateTotal() { }
}
```

---

## 4. Use Case

### 4.1 구조

```
□ Use Case가 하나의 비즈니스 플로우만 처리하는가?
□ Command/Result 패턴을 사용하는가?
□ 트랜잭션 경계가 명확한가?
□ 비즈니스 로직이 도메인에 위임되었는가?
```

**검증 예시**:
```java
// ✅ 올바른 Use Case 구조
class PlaceOrderUseCase {
    private final OrderRepository orderRepository;
    private final EventPublisher eventPublisher;

    @Transactional
    public PlaceOrderResult execute(PlaceOrderCommand command) {
        // 1. 조회
        Order order = orderRepository.findById(command.getOrderId());

        // 2. 도메인 로직 실행 (위임)
        order.place();

        // 3. 저장
        orderRepository.save(order);

        // 4. 이벤트 발행
        eventPublisher.publish(new OrderPlacedEvent(order.getId()));

        // 5. 결과 반환
        return PlaceOrderResult.success(order.getId());
    }
}
```

### 4.2 Port 인터페이스

```
□ Port가 Application Layer에 정의되어 있는가?
□ Port 이름이 도메인 관점인가? (기술 관점 아님)
□ Port가 도메인 타입을 사용하는가?
```

**검증 예시**:
```java
// ✅ 올바른 Port
package com.shop.application.port;

interface OrderRepository {  // 도메인 관점 이름
    Order findById(OrderId id);  // 도메인 타입 사용
    void save(Order order);
}

// ❌ 잘못된 Port
package com.shop.infrastructure;  // 잘못된 위치

interface OrderJpaRepository {  // 기술 관점 이름
    OrderEntity findById(Long id);  // 기술 타입 사용
}
```

---

## 5. Infrastructure

### 5.1 Adapter

```
□ Adapter가 Port를 구현하는가?
□ 기술 세부사항이 Adapter 내부에 캡슐화되었는가?
□ 도메인 모델과 Infrastructure 모델이 분리되어 있는가?
```

**검증 예시**:
```java
// ✅ 올바른 Adapter
class OrderJpaAdapter implements OrderRepository {  // Port 구현
    private final OrderJpaRepository jpaRepository;

    @Override
    public Order findById(OrderId id) {
        OrderEntity entity = jpaRepository.findById(id.getValue())
            .orElseThrow(() -> new OrderNotFoundException(id));

        return toDomain(entity);  // Entity → Domain 변환
    }

    @Override
    public void save(Order order) {
        OrderEntity entity = toEntity(order);  // Domain → Entity 변환
        jpaRepository.save(entity);
    }

    // 변환 로직은 Adapter 내부에
    private Order toDomain(OrderEntity entity) { }
    private OrderEntity toEntity(Order order) { }
}
```

### 5.2 도메인 오염 방지

```
□ JPA 어노테이션이 도메인 모델에 없는가?
□ 프레임워크 의존성이 도메인에 없는가?
□ 도메인 모델이 순수 Java인가?
```

**검증 예시**:
```java
// ❌ 도메인 오염
package com.shop.domain;
import javax.persistence.*;

@Entity
@Table(name = "orders")
class Order {  // JPA 어노테이션으로 오염
    @Id
    @GeneratedValue
    private Long id;
}

// ✅ 순수 도메인
package com.shop.domain;

class Order {  // 순수 Java
    private final OrderId id;
    private OrderStatus status;
}

// Infrastructure에 별도 Entity
package com.shop.infrastructure;
import javax.persistence.*;

@Entity
@Table(name = "orders")
class OrderEntity {
    @Id
    private String id;
    // ...
}
```

---

## 6. 코드 품질

### 6.1 명명

```
□ 이름이 의도를 명확히 표현하는가?
□ 도메인 용어를 사용하는가?
□ 기술 용어를 피했는가?
□ 약어를 피했는가?
```

**검증 예시**:
```java
// ❌ 불명확한 이름
class OrderService {
    public void process(OrderData data) { }
    public void updateStatus(String status) { }
}

// ✅ 명확한 이름
class PlaceOrderUseCase {
    public PlaceOrderResult execute(PlaceOrderCommand command) { }
}

class Order {
    public void place() { }
    public void cancel() { }
}
```

### 6.2 메서드 크기

```
□ 메서드가 한 가지 일만 하는가?
□ 메서드 길이가 적절한가? (20줄 이하 권장)
□ 추상화 수준이 일관적인가?
□ 중첩 depth가 깊지 않은가? (2단계 이하 권장)
```

**검증 예시**:
```java
// ❌ 너무 긴 메서드
public void placeOrder(OrderRequest request) {
    if (request == null) { }
    if (request.getUserId() == null) { }
    // ... 100줄
}

// ✅ 적절한 크기
public void place() {
    validateCanPlace();
    markAsPlaced();
    registerEvent();
}

private void validateCanPlace() { }
private void markAsPlaced() { }
private void registerEvent() { }
```

### 6.3 불변성

```
□ 값 객체가 불변인가?
□ 필드가 final로 선언되었는가?
□ 방어적 복사를 사용하는가?
□ 컬렉션을 안전하게 반환하는가?
```

**검증 예시**:
```java
// ✅ 불변 값 객체
class Money {
    private final BigDecimal amount;  // final
    private final Currency currency;  // final

    public Money add(Money other) {
        return new Money(  // 새 인스턴스 반환
            this.amount.add(other.amount),
            this.currency
        );
    }
}

// ✅ 안전한 컬렉션 반환
class Order {
    private final List<OrderItem> items = new ArrayList<>();

    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);  // 방어적 복사
    }
}
```

---

## 7. 테스트

### 7.1 테스트 가능성

```
□ 클래스가 테스트하기 쉬운가?
□ 의존성을 Mock으로 대체할 수 있는가?
□ 부작용이 최소화되었는가?
□ 테스트가 독립적으로 실행되는가?
```

### 7.2 테스트 커버리지

```
□ 도메인 로직에 단위 테스트가 있는가?
□ Use Case에 통합 테스트가 있는가?
□ 엣지 케이스를 테스트하는가?
□ 예외 상황을 테스트하는가?
```

**검증 예시**:
```java
// ✅ 테스트 가능한 설계
class PlaceOrderUseCase {
    private final OrderRepository repository;  // Mock 가능

    PlaceOrderUseCase(OrderRepository repository) {
        this.repository = repository;
    }
}

// 테스트
@Test
void should_place_order_successfully() {
    // given
    OrderRepository mockRepository = mock(OrderRepository.class);
    PlaceOrderUseCase useCase = new PlaceOrderUseCase(mockRepository);

    // when
    Result result = useCase.execute(command);

    // then
    assertThat(result.isSuccess()).isTrue();
}
```

---

## 8. 성능 고려사항

### 8.1 N+1 문제

```
□ 연관 관계 조회 시 N+1 문제가 없는가?
□ Fetch Join 또는 Batch Size가 설정되었는가?
□ 불필요한 데이터 조회가 없는가?
```

### 8.2 트랜잭션

```
□ 트랜잭션 범위가 적절한가?
□ 긴 트랜잭션이 없는가?
□ Read-only 트랜잭션을 사용했는가?
```

---

## 9. 보안

### 9.1 입력 검증

```
□ 사용자 입력이 검증되는가?
□ SQL Injection 방어가 되어 있는가?
□ XSS 방어가 되어 있는가?
```

### 9.2 권한 검사

```
□ 권한 검사가 적절한 위치에 있는가?
□ 민감한 정보가 로그에 남지 않는가?
□ 암호화가 필요한 데이터를 보호하는가?
```

---

## 10. 문서화

### 10.1 코드 주석

```
□ Why(왜)를 설명하는 주석이 있는가?
□ 복잡한 알고리즘을 설명하는가?
□ 당연한 내용의 주석은 제거되었는가?
□ API 문서(JavaDoc)가 있는가?
```

**검증 예시**:
```java
// ❌ 불필요한 주석
// 사용자 ID를 가져온다
public UserId getUserId() {
    return userId;
}

// ✅ 필요한 주석
// 배송비 무료 임계값은 비즈니스 정책에 따라 변경될 수 있음
// 2024년 1월 기준: 50,000원
private static final Money FREE_SHIPPING_THRESHOLD = Money.won(50_000);

/**
 * 주문을 확정한다.
 *
 * <p>전제조건:
 * - 주문 상태가 DRAFT여야 함
 * - 주문 항목이 1개 이상이어야 함
 *
 * <p>사후조건:
 * - 주문 상태가 PLACED로 변경됨
 * - OrderPlacedEvent가 발행됨
 *
 * @throws EmptyOrderException 주문 항목이 없는 경우
 * @throws OrderAlreadyPlacedException 이미 확정된 주문인 경우
 */
public void place() {
    // ...
}
```

---

## 리뷰 우선순위

### P0 (필수 수정)
- 아키텍처 원칙 위반
- 보안 취약점
- 치명적 버그
- 데이터 손실 가능성

### P1 (강력 권장)
- SRP 위반
- 의존성 방향 문제
- 테스트 부족
- 성능 이슈

### P2 (권장)
- 명명 개선
- 코드 중복
- 주석 개선
- 리팩토링 제안

### P3 (선택)
- 코드 스타일
- 사소한 개선
- 최적화 제안

---

## 자동 검증 스크립트

```bash
# ArchUnit 테스트 실행
./gradlew test --tests "*ArchitectureTest"

# 정적 분석
./gradlew checkstyleMain
./gradlew pmdMain
./gradlew spotbugsMain

# 커버리지 확인
./gradlew jacocoTestReport
```

---

## 리뷰 체크리스트 사용법

1. **리뷰 전**: 작성자가 self-review
2. **리뷰 중**: 리뷰어가 체크리스트 확인
3. **리뷰 후**: 개선 사항 정리
4. **팀 회고**: 반복되는 이슈 패턴 파악
