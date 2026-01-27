# 의존성 관리 가이드

## 목적

클린 아키텍처의 핵심인 의존성 방향 제어와 관리 전략.

---

## 의존성 원칙

### Dependency Rule (의존성 규칙)

**핵심**: 소스 코드 의존성은 반드시 안쪽(도메인)으로만 향해야 한다

```
┌──────────────────────────────────────┐
│      Presentation (UI/API)           │  ← 불안정
├──────────────────────────────────────┤
│         Application (Use Case)        │
├──────────────────────────────────────┤
│            Domain (Rules)             │  ← 안정
├──────────────────────────────────────┤
│       Infrastructure (Tech)           │  ← 불안정
└──────────────────────────────────────┘

의존성 방향:
Presentation ──→ Application ──→ Domain ←── Infrastructure
                                    ↑
                                 (절대 의존 불가)
```

**금지 사항**:
```java
// ❌ Domain이 Infrastructure 의존
package com.shop.domain;

import com.shop.infrastructure.OrderJpaRepository;  // 금지!
import javax.persistence.Entity;                      // 금지!

@Entity  // JPA 어노테이션 금지
class Order {
    private OrderJpaRepository repository;  // Infrastructure 의존 금지
}

// ❌ Domain이 Application 의존
package com.shop.domain;

import com.shop.application.PlaceOrderCommand;  // 금지!

class Order {
    public void place(PlaceOrderCommand command) { }
}
```

### Dependency Inversion Principle (DIP)

**원칙**:
1. 상위 수준 모듈은 하위 수준 모듈에 의존하지 않는다. 둘 다 추상화에 의존한다
2. 추상화는 구체적 사항에 의존하지 않는다. 구체적 사항은 추상화에 의존한다

```java
// ✅ DIP 적용
// Application Layer (상위 수준)
interface OrderRepository {  // 추상화
    Order findById(OrderId id);
    void save(Order order);
}

class PlaceOrderUseCase {
    private final OrderRepository repository;  // 추상화 의존

    public Result execute(Command command) {
        Order order = repository.findById(command.getOrderId());
        // ...
    }
}

// Infrastructure Layer (하위 수준)
class OrderJpaRepository implements OrderRepository {  // 구체화
    // JPA 구현 세부사항
    @Override
    public Order findById(OrderId id) {
        // ...
    }
}

// ❌ DIP 위반
class PlaceOrderUseCase {
    private final OrderJpaRepository repository;  // 구체 클래스 의존

    public Result execute(Command command) {
        // Jpa 구현에 강하게 결합됨
    }
}
```

---

## 의존성 측정 지표

### 1. Afferent Coupling (Ca) - 들어오는 의존성

**정의**: 이 컴포넌트에 의존하는 외부 클래스의 수

```java
// Domain Layer
class Order { }  // Ca = 3 (PlaceOrderUseCase, CancelOrderUseCase, OrderJpaRepository가 의존)

// Application Layer
class PlaceOrderUseCase { }  // Ca = 1 (OrderController가 의존)

// Infrastructure Layer
class OrderJpaRepository { }  // Ca = 0 (아무도 의존하지 않음, DI 컨테이너만 알고 있음)
```

### 2. Efferent Coupling (Ce) - 나가는 의존성

**정의**: 이 컴포넌트가 의존하는 외부 클래스의 수

```java
// Domain Layer
class Order { }  // Ce = 2 (OrderId, Money에 의존)

// Application Layer
class PlaceOrderUseCase {  // Ce = 3
    private final OrderRepository repository;     // 1
    private final PaymentGateway paymentGateway;  // 2
    private final EventPublisher publisher;       // 3
}

// Infrastructure Layer
class OrderJpaRepository {  // Ce = 5
    // JPA, EntityManager, Order, OrderId, OrderEntity 등에 의존
}
```

### 3. Instability (I) - 불안정성

**공식**: `I = Ce / (Ca + Ce)`

**값 범위**: 0 (완전 안정) ~ 1 (완전 불안정)

```
I = 0 (안정적)
- 많은 클래스가 의존 (Ca 높음)
- 외부 의존 없음 (Ce 낮음)
- 변경이 어려움
- 예: Domain entities

I = 1 (불안정)
- 의존하는 클래스 없음 (Ca 낮음)
- 많은 외부 의존 (Ce 높음)
- 변경이 쉬움
- 예: Infrastructure adapters

이상적:
Domain: I = 0 ~ 0.2  (매우 안정)
Application: I = 0.3 ~ 0.5 (중간)
Infrastructure: I = 0.6 ~ 1.0 (불안정)
```

### 4. Abstractness (A) - 추상화 정도

**공식**: `A = 추상 클래스/인터페이스 수 / 전체 클래스 수`

**값 범위**: 0 (완전 구체) ~ 1 (완전 추상)

```
A = 0 (구체적)
- 추상화 없음
- 예: Infrastructure 구현 클래스

A = 1 (추상적)
- 모두 인터페이스
- 예: Application Layer의 Port

이상적:
Domain: A = 0.1 ~ 0.3  (주로 구체 클래스, 일부 인터페이스)
Application: A = 0.3 ~ 0.5 (절반은 Port 인터페이스)
Infrastructure: A = 0 ~ 0.2 (대부분 구체 클래스)
```

### 5. Distance from Main Sequence (D)

**공식**: `D = |A + I - 1|`

**의미**: Main Sequence(A + I = 1)로부터의 거리

```
A (추상화)
 1 │   Zone of Uselessness
   │   ╱
   │  ╱ Main Sequence
   │ ╱  (A + I = 1)
   │╱
 0 └────────────────────> 1 I (불안정)
      Zone of Pain

이상적: D = 0 ~ 0.1 (Main Sequence에 가까움)
경고: D > 0.3
위험: D > 0.5
```

---

## 의존성 관리 전략

### 1. 인터페이스 배치 전략

**원칙**: 인터페이스는 클라이언트(사용자) 쪽에 위치한다

```java
// ✅ 올바른 배치
// Application Layer
package com.shop.application.port;

interface OrderRepository {  // Application에 정의
    Order findById(OrderId id);
}

interface PaymentGateway {  // Application에 정의
    PaymentResult process(Payment payment);
}

// Infrastructure Layer
package com.shop.infrastructure.persistence;

class OrderJpaRepository implements OrderRepository {  // Port 구현
    // ...
}

package com.shop.infrastructure.payment;

class StripePaymentGateway implements PaymentGateway {  // Port 구현
    // ...
}

// ❌ 잘못된 배치
// Infrastructure Layer에 인터페이스 정의
package com.shop.infrastructure;

interface OrderRepository { }  // 잘못된 위치

class OrderJpaRepository implements OrderRepository { }
```

### 2. Adapter 패턴

**목적**: Infrastructure를 Domain/Application으로부터 격리

```java
// Application Layer - Port 정의
interface NotificationService {
    void notify(UserId userId, String message);
}

// Infrastructure Layer - Adapter 구현
class EmailNotificationAdapter implements NotificationService {
    private final SendGridClient sendGridClient;  // 외부 라이브러리

    @Override
    public void notify(UserId userId, String message) {
        // SendGrid API를 Domain 개념으로 변환
        User user = userRepository.findById(userId);
        Email email = Email.builder()
            .to(user.getEmail())
            .subject("알림")
            .content(message)
            .build();

        sendGridClient.send(email);
    }
}

// Use Case는 Port만 알고 있음
class PlaceOrderUseCase {
    private final NotificationService notificationService;  // Port

    public Result execute(Command command) {
        // ...
        notificationService.notify(order.getUserId(), "주문 완료");
    }
}
```

### 3. Anti-Corruption Layer (ACL)

**목적**: 레거시 시스템이나 외부 시스템으로부터 도메인 보호

```java
// 외부 레거시 시스템
class LegacyOrderSystem {
    public LegacyOrderData getOrder(String orderId) {
        // 복잡하고 일관성 없는 데이터 구조
    }
}

// ACL - 레거시를 도메인 언어로 변환
class LegacyOrderAdapter implements OrderRepository {
    private final LegacyOrderSystem legacySystem;

    @Override
    public Order findById(OrderId id) {
        // 1. 레거시 시스템 호출
        LegacyOrderData legacyData = legacySystem.getOrder(id.getValue());

        // 2. 도메인 모델로 변환
        return Order.builder()
            .id(OrderId.of(legacyData.getOrderNo()))
            .userId(UserId.of(legacyData.getUserCode()))
            .status(mapStatus(legacyData.getState()))
            .totalAmount(Money.won(legacyData.getPrice()))
            .build();
    }

    private OrderStatus mapStatus(String legacyStatus) {
        // 레거시 상태 코드를 도메인 상태로 매핑
        return switch (legacyStatus) {
            case "10" -> OrderStatus.PENDING;
            case "20" -> OrderStatus.PAID;
            case "30" -> OrderStatus.SHIPPED;
            default -> OrderStatus.UNKNOWN;
        };
    }
}
```

### 4. Plugin Architecture

**목적**: 핵심 비즈니스 로직과 플러그인 분리

```
Core (Domain + Application)
   ↑
   │ implements
   │
Plugins (Infrastructure)
   ├─ JPA Plugin
   ├─ Stripe Plugin
   ├─ SendGrid Plugin
   └─ Redis Plugin
```

```java
// Core - Port 정의
interface PaymentGateway {
    PaymentResult process(Payment payment);
}

// Plugin 1
class StripePlugin implements PaymentGateway {
    // Stripe 구현
}

// Plugin 2
class TossPlugin implements PaymentGateway {
    // Toss 구현
}

// Plugin 3
class PaypalPlugin implements PaymentGateway {
    // Paypal 구현
}

// Configuration에서 플러그인 교체
@Configuration
class PaymentConfig {
    @Bean
    public PaymentGateway paymentGateway() {
        if (profile.equals("production")) {
            return new StripePlugin();
        } else {
            return new TossPlugin();
        }
    }
}
```

---

## 순환 의존성 해결

### 문제: 순환 의존성

```java
// ❌ 순환 의존성
class Order {
    private Payment payment;  // Order → Payment

    public void place() {
        payment.process(this);  // Payment에게 Order 전달
    }
}

class Payment {
    private Order order;  // Payment → Order (순환!)

    public void process(Order order) {
        this.order = order;
    }
}
```

### 해결책 1: 의존성 방향 통일

```java
// ✅ 의존성 방향 통일 (Order → Payment만)
class Order {
    private PaymentId paymentId;  // ID로만 참조

    public void place(PaymentGateway gateway) {
        Payment payment = gateway.findById(this.paymentId);
        payment.process(this.totalAmount);
    }
}

class Payment {
    // Order에 대한 참조 없음

    public void process(Money amount) {
        // ...
    }
}
```

### 해결책 2: 이벤트로 순환 제거

```java
// ✅ 이벤트 사용
class Order {
    public void place() {
        // ...
        registerEvent(new OrderPlacedEvent(this.id, this.totalAmount));
    }
}

class PaymentEventHandler {
    private final PaymentGateway paymentGateway;

    @EventListener
    public void handle(OrderPlacedEvent event) {
        Payment payment = Payment.create(event.getOrderId(), event.getAmount());
        paymentGateway.process(payment);
    }
}
```

### 해결책 3: 중간 객체 도입

```java
// ✅ 중간 객체 (Mediator)
class OrderPaymentMediator {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;

    public void processOrderPayment(OrderId orderId) {
        Order order = orderRepository.findById(orderId);
        Payment payment = Payment.create(orderId, order.getTotalAmount());
        paymentGateway.process(payment);
        order.markAsPaid();
    }
}
```

---

## 의존성 주입 (DI) 패턴

### Constructor Injection (권장)

```java
// ✅ Constructor Injection
class PlaceOrderUseCase {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final EventPublisher eventPublisher;

    // 모든 의존성을 생성자로 주입
    public PlaceOrderUseCase(
            OrderRepository orderRepository,
            PaymentGateway paymentGateway,
            EventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.eventPublisher = eventPublisher;
    }
}

// 장점:
// 1. 불변성 보장 (final)
// 2. 필수 의존성 명확
// 3. 테스트 용이
// 4. 순환 의존성 컴파일 타임 발견
```

### Field Injection (비권장)

```java
// ❌ Field Injection
class PlaceOrderUseCase {
    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private PaymentGateway paymentGateway;

    // 단점:
    // 1. 불변성 보장 불가
    // 2. 테스트 어려움
    // 3. 순환 의존성 런타임 발견
    // 4. 의존성 숨김
}
```

### Setter Injection (선택적 의존성에만)

```java
// ⚠️ Setter Injection (선택적 의존성에만)
class PlaceOrderUseCase {
    private final OrderRepository orderRepository;
    private NotificationService notificationService;  // 선택적

    public PlaceOrderUseCase(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;  // 필수
    }

    // 선택적 의존성은 setter로
    public void setNotificationService(NotificationService service) {
        this.notificationService = service;
    }
}
```

---

## 의존성 검증 도구

### 1. ArchUnit 테스트

```java
@Test
void domainLayerShouldNotDependOnInfrastructure() {
    noClasses()
        .that().resideInPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInPackage("..infrastructure..")
        .check(classes);
}

@Test
void domainLayerShouldNotDependOnApplication() {
    noClasses()
        .that().resideInPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInPackage("..application..")
        .check(classes);
}

@Test
void infrastructureShouldImplementPorts() {
    classes()
        .that().resideInPackage("..infrastructure..")
        .and().areNotInterfaces()
        .should().implement(interfaceThat().resideInPackage("..application.port.."))
        .check(classes);
}

@Test
void noCircularDependencies() {
    slices()
        .matching("com.shop.(*)..")
        .should().beFreeOfCycles()
        .check(classes);
}
```

### 2. Maven Dependency Plugin

```bash
# 의존성 트리 분석
mvn dependency:tree

# 순환 의존성 검사
mvn dependency:analyze

# 불필요한 의존성 찾기
mvn dependency:analyze-only
```

### 3. JDepend

```java
JDepend jdepend = new JDepend();
jdepend.addDirectory("/path/to/classes");
Collection packages = jdepend.analyze();

for (JavaPackage pkg : packages) {
    System.out.println("Package: " + pkg.getName());
    System.out.println("  Afferent Coupling (Ca): " + pkg.afferentCoupling());
    System.out.println("  Efferent Coupling (Ce): " + pkg.efferentCoupling());
    System.out.println("  Instability (I): " + pkg.instability());
    System.out.println("  Abstractness (A): " + pkg.abstractness());
    System.out.println("  Distance (D): " + pkg.distance());
}
```

---

## 체크리스트

### 의존성 방향 검증

- [ ] Domain Layer가 다른 레이어에 의존하지 않는가?
- [ ] Application Layer가 Infrastructure를 직접 의존하지 않는가?
- [ ] Port 인터페이스는 Application Layer에 정의되어 있는가?
- [ ] Infrastructure는 Port를 구현하는가?

### 의존성 품질 검증

- [ ] 구체 클래스 대신 인터페이스에 의존하는가?
- [ ] 순환 의존성이 없는가?
- [ ] 불필요한 의존성이 없는가?
- [ ] 의존성 수가 적절한가? (5개 이하 권장)

### 아키텍처 메트릭

- [ ] Domain의 불안정성(I)이 0.2 이하인가?
- [ ] Infrastructure의 불안정성(I)이 0.6 이상인가?
- [ ] Main Sequence 거리(D)가 0.3 이하인가?
- [ ] 패키지 간 순환 의존성이 없는가?
