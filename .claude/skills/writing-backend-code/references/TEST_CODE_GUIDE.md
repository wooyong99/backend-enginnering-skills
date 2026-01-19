# 테스트 코드 작성 가이드

> **목적**: 결정적이고 빠르며 유지보수 가능한 테스트 작성  
> **대상**: 백엔드 개발자 및 AI 개발 어시스턴트  
> **원칙**: FIRST 원칙 + Test Pyramid

---

## 📋 목차

1. [테스트 기본 원칙](#테스트-기본-원칙)
2. [테스트 피라미드](#테스트-피라미드)
3. [단위 테스트 작성](#단위-테스트-작성)
4. [통합 테스트 작성](#통합-테스트-작성)
5. [테스트 더블 활용](#테스트-더블-활용)
6. [테스트 데이터 관리](#테스트-데이터-관리)
7. [테스트 성능 최적화](#테스트-성능-최적화)
8. [안티패턴과 해결책](#안티패턴과-해결책)

---

## 테스트 기본 원칙

### FIRST 원칙

**F - Fast (빠른)**

```
원칙: 테스트는 빠르게 실행되어야 한다.

목표:
- 단위 테스트: < 100ms
- 통합 테스트: < 1s
- E2E 테스트: < 10s

✅ GOOD: 빠른 테스트
@Test
void calculatePrice() {
    // Given
    Order order = OrderFixture.create();

    // When
    Money price = order.calculateTotal();

    // Then
    assertThat(price).isEqualTo(Money.won(10000));
}
// 실행 시간: 5ms

❌ BAD: 느린 테스트
@Test
void calculatePrice() {
    // Given
    Order order = orderRepository.save(OrderFixture.create());  // DB 접근
    Thread.sleep(1000);  // 불필요한 대기

    // When
    Money price = order.calculateTotal();

    // Then
    assertThat(price).isEqualTo(Money.won(10000));
}
// 실행 시간: 1050ms
```

**I - Independent/Isolated (독립적)**

```
원칙: 테스트는 서로 독립적이어야 한다.

✅ GOOD: 독립적인 테스트
@Test
void createOrder() {
    Order order = Order.create(command);
    assertThat(order.getStatus()).isEqualTo(OrderStatus.CREATED);
}

@Test
void cancelOrder() {
    Order order = Order.create(command);
    order.cancel();
    assertThat(order.getStatus()).isEqualTo(OrderStatus.CANCELLED);
}
// 각 테스트가 독립적으로 실행됨

❌ BAD: 의존적인 테스트
private Order sharedOrder;  // 공유 상태

@Test
void test1_createOrder() {
    sharedOrder = Order.create(command);  // 상태 변경
}

@Test
void test2_cancelOrder() {
    sharedOrder.cancel();  // test1에 의존
}
// test2는 test1 이후에만 성공
```

**R - Repeatable (반복 가능)**

```
원칙: 어떤 환경에서든 반복 가능해야 한다.

✅ GOOD: 결정적인 테스트
@Test
void calculateDiscount() {
    // Given
    LocalDateTime fixedTime = LocalDateTime.of(2024, 1, 1, 0, 0);
    Clock fixedClock = Clock.fixed(
        fixedTime.toInstant(ZoneOffset.UTC),
        ZoneOffset.UTC
    );

    DiscountPolicy policy = new DiscountPolicy(fixedClock);

    // When
    Money discount = policy.calculate(order);

    // Then
    assertThat(discount).isEqualTo(Money.won(1000));
}
// 항상 같은 결과

❌ BAD: 비결정적인 테스트
@Test
void calculateDiscount() {
    // Given
    DiscountPolicy policy = new DiscountPolicy();

    // When
    Money discount = policy.calculate(order);  // 현재 시간에 따라 다름

    // Then
    assertThat(discount).isEqualTo(Money.won(1000));  // 때로는 실패
}
// 실행 시점에 따라 결과가 달라짐
```

**S - Self-Validating (자가 검증)**

```
원칙: 테스트는 스스로 성공/실패를 판단해야 한다.

✅ GOOD: 자가 검증
@Test
void createOrder() {
    Order order = Order.create(command);

    assertThat(order.getStatus()).isEqualTo(OrderStatus.CREATED);
    assertThat(order.getItems()).hasSize(2);
}
// 명확한 검증

❌ BAD: 수동 검증
@Test
void createOrder() {
    Order order = Order.create(command);

    System.out.println(order);  // 콘솔 출력 확인 필요
}
// 사람이 직접 확인해야 함
```

**T - Timely (적시에)**

```
원칙: 테스트는 프로덕션 코드를 작성하기 직전에 작성한다.

TDD 사이클:
1. Red: 실패하는 테스트 작성
2. Green: 테스트를 통과하는 최소한의 코드 작성
3. Refactor: 코드 개선

✅ GOOD: TDD 방식
// 1. 테스트 먼저 작성
@Test
void cancelOrder() {
    Order order = Order.create(command);
    order.cancel();
    assertThat(order.getStatus()).isEqualTo(OrderStatus.CANCELLED);
}

// 2. 구현
public void cancel() {
    this.status = OrderStatus.CANCELLED;
}

❌ BAD: 나중에 작성
// 1. 구현 먼저
public void cancel() {
    this.status = OrderStatus.CANCELLED;
    // 복잡한 로직 100줄...
}

// 2. 테스트를 나중에 (또는 안 함)
```

---

## 테스트 피라미드

### 피라미드 구조

```
       E2E
      /   \
     /     \
    /       \
   / 통합 테스트 \
  /             \
 /               \
/   단위 테스트    \
-------------------

비율: 단위 70% : 통합 20% : E2E 10%
속도: 단위 > 통합 > E2E
안정성: 단위 > 통합 > E2E
```

### 레이어별 테스트 전략

**1. 단위 테스트 (Unit Test)**

```
대상:
- Domain Entity
- Value Object
- Domain Service
- Use Case (Port는 Mock)

특징:
- 매우 빠름 (< 100ms)
- 외부 의존성 없음
- 높은 커버리지
- 격리된 테스트

예시:
@Test
void addItem() {
    // Given
    Order order = OrderFixture.create();
    Product product = ProductFixture.create();

    // When
    order.addItem(product, 2);

    // Then
    assertThat(order.getItems()).hasSize(1);
    assertThat(order.getTotalAmount()).isEqualTo(Money.won(20000));
}
```

**2. 통합 테스트 (Integration Test)**

```
대상:
- Repository 구현체
- External API Client
- Event Publisher
- Infrastructure 레이어

특징:
- 중간 속도 (< 1s)
- 실제 외부 시스템 연동
- 설정 필요 (DB, Message Queue 등)

예시:
@SpringBootTest
@Transactional
class OrderRepositoryTest {

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void saveOrder() {
        // Given
        Order order = OrderFixture.create();

        // When
        Order saved = orderRepository.save(order);

        // Then
        assertThat(saved.getId()).isNotNull();

        Order found = orderRepository.findById(saved.getId());
        assertThat(found).isEqualTo(saved);
    }
}
```

**3. E2E 테스트 (End-to-End Test)**

```
대상:
- API 엔드포인트
- 전체 플로우

특징:
- 느림 (< 10s)
- 실제 시나리오 검증
- 최소한으로 유지

예시:
@SpringBootTest(webEnvironment = RANDOM_PORT)
class OrderApiE2ETest {

    @Autowired
    private TestRestTemplate restTemplate;

    @Test
    void createAndCancelOrder() {
        // Given
        CreateOrderRequest request = CreateOrderRequestFixture.create();

        // When: 주문 생성
        ResponseEntity<OrderResponse> createResponse =
            restTemplate.postForEntity("/api/orders", request, OrderResponse.class);

        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        String orderId = createResponse.getBody().getOrderId();

        // When: 주문 취소
        ResponseEntity<Void> cancelResponse =
            restTemplate.postForEntity("/api/orders/" + orderId + "/cancel",
                null, Void.class);

        // Then
        assertThat(cancelResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

---

## 단위 테스트 작성

### Domain Entity 테스트

**기본 패턴**:

```java
class OrderTest {

    @Test
    @DisplayName("주문 생성 시 상태는 CREATED여야 한다")
    void createOrder() {
        // Given
        CreateOrderCommand command = CreateOrderCommand.builder()
            .userId(UserId.of("user-123"))
            .items(List.of(
                OrderItemCommand.of(ProductId.of("prod-1"), 2)
            ))
            .build();

        // When
        Order order = Order.create(command);

        // Then
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CREATED);
        assertThat(order.getUserId()).isEqualTo(UserId.of("user-123"));
        assertThat(order.getItems()).hasSize(1);
    }

    @Test
    @DisplayName("배송 시작된 주문은 취소할 수 없다")
    void cannotCancelShippedOrder() {
        // Given
        Order order = OrderFixture.createShipped();

        // When & Then
        assertThatThrownBy(() -> order.cancel())
            .isInstanceOf(OrderAlreadyShippedException.class)
            .hasMessage("배송 시작된 주문은 취소할 수 없습니다");
    }

    @Test
    @DisplayName("주문 금액은 상품 가격의 합이다")
    void calculateTotalAmount() {
        // Given
        Order order = OrderFixture.create();
        order.addItem(ProductFixture.create(Money.won(10000)), 2);
        order.addItem(ProductFixture.create(Money.won(5000)), 3);

        // When
        Money total = order.calculateTotal();

        // Then
        assertThat(total).isEqualTo(Money.won(35000));
    }
}
```

**Value Object 테스트**:

```java
class MoneyTest {

    @Test
    @DisplayName("금액 덧셈")
    void add() {
        // Given
        Money money1 = Money.won(10000);
        Money money2 = Money.won(5000);

        // When
        Money result = money1.add(money2);

        // Then
        assertThat(result).isEqualTo(Money.won(15000));
    }

    @Test
    @DisplayName("다른 통화는 연산할 수 없다")
    void cannotAddDifferentCurrency() {
        // Given
        Money krw = Money.of(10000, Currency.getInstance("KRW"));
        Money usd = Money.of(100, Currency.getInstance("USD"));

        // When & Then
        assertThatThrownBy(() -> krw.add(usd))
            .isInstanceOf(DifferentCurrencyException.class);
    }

    @Test
    @DisplayName("값이 같으면 동등하다")
    void equality() {
        // Given
        Money money1 = Money.won(10000);
        Money money2 = Money.won(10000);

        // Then
        assertThat(money1).isEqualTo(money2);
        assertThat(money1.hashCode()).isEqualTo(money2.hashCode());
    }
}
```

### Use Case 테스트

**Mock을 활용한 격리 테스트**:

```java
class CreateOrderUseCaseTest {

    private CreateOrderUseCase useCase;
    private OrderRepository orderRepository;
    private UserRepository userRepository;
    private InventoryChecker inventoryChecker;
    private EventPublisher eventPublisher;

    @BeforeEach
    void setUp() {
        orderRepository = mock(OrderRepository.class);
        userRepository = mock(UserRepository.class);
        inventoryChecker = mock(InventoryChecker.class);
        eventPublisher = mock(EventPublisher.class);

        useCase = new CreateOrderUseCase(
            orderRepository,
            userRepository,
            inventoryChecker,
            eventPublisher
        );
    }

    @Test
    @DisplayName("주문 생성 성공")
    void createOrderSuccess() {
        // Given
        CreateOrderCommand command = CreateOrderCommandFixture.create();

        User user = UserFixture.create();
        when(userRepository.findById(command.getUserId()))
            .thenReturn(user);

        when(inventoryChecker.checkAvailability(any()))
            .thenReturn(InventoryCheckResult.available());

        when(orderRepository.save(any(Order.class)))
            .thenAnswer(invocation -> invocation.getArgument(0));

        // When
        OrderResult result = useCase.execute(command);

        // Then
        assertThat(result.getStatus()).isEqualTo(OrderStatus.CREATED);

        verify(orderRepository).save(any(Order.class));
        verify(eventPublisher).publish(any(OrderCreatedEvent.class));
    }

    @Test
    @DisplayName("재고 부족 시 주문 생성 실패")
    void createOrderFailWhenInsufficientStock() {
        // Given
        CreateOrderCommand command = CreateOrderCommandFixture.create();

        when(userRepository.findById(command.getUserId()))
            .thenReturn(UserFixture.create());

        when(inventoryChecker.checkAvailability(any()))
            .thenReturn(InventoryCheckResult.insufficient());

        // When & Then
        assertThatThrownBy(() -> useCase.execute(command))
            .isInstanceOf(InsufficientStockException.class);

        verify(orderRepository, never()).save(any());
        verify(eventPublisher, never()).publish(any());
    }
}
```

### Domain Service 테스트

```java
class PriceCalculatorTest {

    private PriceCalculator calculator;

    @BeforeEach
    void setUp() {
        calculator = new PriceCalculator();
    }

    @Test
    @DisplayName("VIP 회원은 15% 할인")
    void vipDiscount() {
        // Given
        Order order = OrderFixture.withAmount(Money.won(100000));
        MemberGrade grade = MemberGrade.VIP;

        // When
        Money finalPrice = calculator.calculateFinalPrice(
            order, grade, List.of()
        );

        // Then
        assertThat(finalPrice).isEqualTo(Money.won(85000));
    }

    @Test
    @DisplayName("50,000원 이상 구매 시 무료 배송")
    void freeShipping() {
        // Given
        Order order = OrderFixture.withAmount(Money.won(50000));

        // When
        Money shippingFee = calculator.calculateShippingFee(order);

        // Then
        assertThat(shippingFee).isEqualTo(Money.ZERO);
    }

    @ParameterizedTest
    @CsvSource({
        "10000, 3000",
        "30000, 3000",
        "49999, 3000",
        "50000, 0",
        "100000, 0"
    })
    @DisplayName("배송비 계산 테스트")
    void calculateShippingFee(long amount, long expectedFee) {
        // Given
        Order order = OrderFixture.withAmount(Money.won(amount));

        // When
        Money shippingFee = calculator.calculateShippingFee(order);

        // Then
        assertThat(shippingFee).isEqualTo(Money.won(expectedFee));
    }
}
```

---

## 통합 테스트 작성

### Repository 통합 테스트

**기본 패턴**:

```java
@SpringBootTest
@Transactional
class OrderRepositoryIntegrationTest {

    @Autowired
    private OrderRepository orderRepository;

    @Test
    @DisplayName("주문 저장 및 조회")
    void saveAndFind() {
        // Given
        Order order = OrderFixture.create();

        // When
        Order saved = orderRepository.save(order);

        // Then
        assertThat(saved.getId()).isNotNull();

        Order found = orderRepository.findById(saved.getId());
        assertThat(found).isNotNull();
        assertThat(found.getUserId()).isEqualTo(order.getUserId());
    }

    @Test
    @DisplayName("사용자 ID로 주문 목록 조회")
    void findByUserId() {
        // Given
        UserId userId = UserId.of("user-123");

        orderRepository.save(OrderFixture.create(userId));
        orderRepository.save(OrderFixture.create(userId));
        orderRepository.save(OrderFixture.create(UserId.of("user-456")));

        // When
        List<Order> orders = orderRepository.findByUserId(userId);

        // Then
        assertThat(orders).hasSize(2);
        assertThat(orders).allMatch(o -> o.getUserId().equals(userId));
    }
}
```

**TestContainers 활용**:

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryTestContainersTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void setProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void saveOrder() {
        // Given
        Order order = OrderFixture.create();

        // When
        Order saved = orderRepository.save(order);

        // Then
        assertThat(saved.getId()).isNotNull();
    }
}
```

### External API Client 테스트

**WireMock 활용**:

```java
@SpringBootTest
class PaymentApiClientIntegrationTest {

    @Autowired
    private PaymentApiClient paymentApiClient;

    private WireMockServer wireMockServer;

    @BeforeEach
    void setUp() {
        wireMockServer = new WireMockServer(8089);
        wireMockServer.start();
        configureFor("localhost", 8089);
    }

    @AfterEach
    void tearDown() {
        wireMockServer.stop();
    }

    @Test
    @DisplayName("결제 승인 성공")
    void approvePaymentSuccess() {
        // Given
        stubFor(post(urlEqualTo("/api/payments/approve"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "transactionId": "txn-123",
                        "status": "APPROVED",
                        "amount": 10000
                    }
                    """)));

        PaymentRequest request = PaymentRequestFixture.create();

        // When
        PaymentResult result = paymentApiClient.approve(request);

        // Then
        assertThat(result.getTransactionId()).isEqualTo("txn-123");
        assertThat(result.getStatus()).isEqualTo(PaymentStatus.APPROVED);

        verify(postRequestedFor(urlEqualTo("/api/payments/approve"))
            .withRequestBody(matchingJsonPath("$.amount", equalTo("10000"))));
    }

    @Test
    @DisplayName("결제 서비스 타임아웃")
    void approvePaymentTimeout() {
        // Given
        stubFor(post(urlEqualTo("/api/payments/approve"))
            .willReturn(aResponse()
                .withFixedDelay(5000)));  // 5초 지연

        PaymentRequest request = PaymentRequestFixture.create();

        // When & Then
        assertThatThrownBy(() -> paymentApiClient.approve(request))
            .isInstanceOf(PaymentTimeoutException.class);
    }
}
```

---

## 테스트 더블 활용

### Test Double 종류

**1. Dummy (더미)**

```java
// 단지 파라미터를 채우기 위한 객체
@Test
void test() {
    User dummy = new User();  // 실제로 사용되지 않음
    service.execute(command, dummy);
}
```

**2. Stub (스텁)**

```java
// 미리 정의된 답변을 반환
class StubUserRepository implements UserRepository {
    @Override
    public User findById(UserId id) {
        return UserFixture.create();  // 항상 같은 값 반환
    }
}

@Test
void test() {
    UserRepository repository = new StubUserRepository();
    // 테스트 진행
}
```

**3. Spy (스파이)**

```java
// 실제 객체를 부분적으로 감시
@Test
void test() {
    OrderRepository repository = spy(new OrderRepositoryImpl());

    // 특정 메서드만 스텁
    when(repository.findById(any())).thenReturn(OrderFixture.create());

    // 실제 메서드 호출 확인
    verify(repository).save(any());
}
```

**4. Mock (모의 객체)**

```java
// 행위 검증이 필요한 경우
@Test
void test() {
    EventPublisher eventPublisher = mock(EventPublisher.class);

    // When
    service.execute(command);

    // Then: 호출 검증
    verify(eventPublisher).publish(any(OrderCreatedEvent.class));
    verify(eventPublisher, times(1)).publish(any());
    verify(eventPublisher, never()).publish(any(OrderCancelledEvent.class));
}
```

**5. Fake (가짜 객체)**

```java
// 간단한 구현체 (실제 동작하지만 프로덕션용은 아님)
class FakeOrderRepository implements OrderRepository {
    private Map<OrderId, Order> storage = new HashMap<>();

    @Override
    public Order save(Order order) {
        storage.put(order.getId(), order);
        return order;
    }

    @Override
    public Order findById(OrderId id) {
        return storage.get(id);
    }
}

@Test
void test() {
    OrderRepository repository = new FakeOrderRepository();
    // 실제 동작하는 저장소처럼 사용
}
```

### Mock 사용 가이드

**적절한 Mock 사용**:

```java
✅ GOOD: Port에 대한 Mock
@Test
void createOrder() {
    // Port 인터페이스는 Mock 사용
    OrderRepository orderRepository = mock(OrderRepository.class);
    InventoryChecker inventoryChecker = mock(InventoryChecker.class);

    when(inventoryChecker.check(any())).thenReturn(true);

    CreateOrderUseCase useCase = new CreateOrderUseCase(
        orderRepository,
        inventoryChecker
    );

    // 테스트 진행
}
```

**부적절한 Mock 사용**:

```java
❌ BAD: Domain Entity Mock
@Test
void test() {
    Order order = mock(Order.class);  // ❌ Entity는 Mock 사용 금지
    when(order.getStatus()).thenReturn(OrderStatus.CREATED);

    // Entity는 실제 객체 사용
    Order order = OrderFixture.create();  // ✅
}
```

---

## 테스트 데이터 관리

### Fixture 패턴

**Object Mother 패턴**:

```java
public class OrderFixture {

    public static Order create() {
        return create(UserId.of("user-123"));
    }

    public static Order create(UserId userId) {
        return Order.builder()
            .id(OrderId.generate())
            .userId(userId)
            .status(OrderStatus.CREATED)
            .createdAt(LocalDateTime.now())
            .build();
    }

    public static Order createWithItems() {
        Order order = create();
        order.addItem(ProductFixture.create(), 2);
        return order;
    }

    public static Order createShipped() {
        Order order = create();
        order.ship();
        return order;
    }

    public static Order withAmount(Money amount) {
        Order order = create();
        // amount에 맞게 상품 추가
        return order;
    }
}
```

**Test Data Builder 패턴**:

```java
public class OrderBuilder {

    private OrderId id = OrderId.generate();
    private UserId userId = UserId.of("user-123");
    private OrderStatus status = OrderStatus.CREATED;
    private List<OrderItem> items = new ArrayList<>();
    private Money totalAmount = Money.ZERO;

    public OrderBuilder id(OrderId id) {
        this.id = id;
        return this;
    }

    public OrderBuilder userId(UserId userId) {
        this.userId = userId;
        return this;
    }

    public OrderBuilder status(OrderStatus status) {
        this.status = status;
        return this;
    }

    public OrderBuilder addItem(Product product, int quantity) {
        OrderItem item = OrderItem.create(product, quantity);
        this.items.add(item);
        this.totalAmount = this.totalAmount.add(item.getSubtotal());
        return this;
    }

    public Order build() {
        return new Order(id, userId, status, items, totalAmount);
    }
}

// 사용
@Test
void test() {
    Order order = new OrderBuilder()
        .userId(UserId.of("user-456"))
        .status(OrderStatus.PAID)
        .addItem(ProductFixture.laptop(), 1)
        .addItem(ProductFixture.mouse(), 2)
        .build();
}
```

### 테스트 데이터 격리

**각 테스트마다 새로운 데이터**:

```java
✅ GOOD: 격리된 데이터
@Test
void test1() {
    Order order = OrderFixture.create();  // 새 인스턴스
    // 테스트
}

@Test
void test2() {
    Order order = OrderFixture.create();  // 새 인스턴스
    // 테스트
}
```

**DB 데이터 격리**:

```java
@SpringBootTest
@Transactional  // 각 테스트 후 롤백
class OrderRepositoryTest {

    @Test
    void test1() {
        orderRepository.save(OrderFixture.create());
        // 테스트 종료 후 롤백
    }

    @Test
    void test2() {
        // 깨끗한 상태에서 시작
        orderRepository.save(OrderFixture.create());
    }
}
```

---

## 테스트 성능 최적화

### 느린 테스트 개선

**문제: DB 접근이 많은 테스트**

```java
❌ BAD: 매번 DB 접근
@SpringBootTest
class SlowTest {

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void test() {
        Order order = orderRepository.save(OrderFixture.create());  // DB 저장
        Order found = orderRepository.findById(order.getId());     // DB 조회
        // ...
    }
}
// 실행 시간: 500ms

✅ GOOD: 순수 객체로 테스트
class FastTest {

    @Test
    void test() {
        Order order = OrderFixture.create();  // 메모리에만
        // 도메인 로직 테스트
    }
}
// 실행 시간: 5ms
```

**문제: Spring Context 로딩이 많은 테스트**

```java
❌ BAD: 매번 Context 로딩
@SpringBootTest  // 전체 컨텍스트 로딩
class SlowIntegrationTest {
    // ...
}

✅ GOOD: 필요한 것만 로딩
@DataJpaTest  // JPA 관련 빈만 로딩
class FastIntegrationTest {
    // ...
}
```

### 병렬 테스트 실행

**JUnit 5 병렬 실행**:

```properties
# junit-platform.properties
junit.jupiter.execution.parallel.enabled = true
junit.jupiter.execution.parallel.mode.default = concurrent
junit.jupiter.execution.parallel.mode.classes.default = concurrent
```

```java
@Execution(ExecutionMode.CONCURRENT)
class ParallelTest {

    @Test
    void test1() {
        // 병렬 실행
    }

    @Test
    void test2() {
        // 병렬 실행
    }
}
```

### 테스트 캐싱

**Spring Context 재사용**:

```java
// 같은 설정의 테스트는 Context 재사용
@SpringBootTest
class Test1 { }

@SpringBootTest  // Context 재사용됨
class Test2 { }

@SpringBootTest(properties = "app.mode=dev")  // 새로운 Context
class Test3 { }
```

---

## 안티패턴과 해결책

### 안티패턴 1: 테스트 간 의존성

**문제**:

```java
❌ BAD: 테스트 순서에 의존
class BadTest {
    private static Order order;

    @Test
    void step1_create() {
        order = Order.create(command);
    }

    @Test
    void step2_cancel() {
        order.cancel();  // step1에 의존
    }
}
```

**해결**:

```java
✅ GOOD: 각 테스트 독립적
class GoodTest {

    @Test
    void createOrder() {
        Order order = Order.create(command);
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CREATED);
    }

    @Test
    void cancelOrder() {
        Order order = OrderFixture.createPaid();  // 필요한 상태 직접 생성
        order.cancel();
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CANCELLED);
    }
}
```

### 안티패턴 2: 비결정적 테스트

**문제**:

```java
❌ BAD: 현재 시간에 의존
@Test
void calculateDiscount() {
    Order order = Order.create(command);
    Money discount = order.calculateDiscount();  // 현재 시간에 따라 다름
    assertThat(discount).isEqualTo(Money.won(1000));
}
```

**해결**:

```java
✅ GOOD: 시간을 주입
@Test
void calculateDiscount() {
    // Given
    Clock fixedClock = Clock.fixed(
        LocalDateTime.of(2024, 1, 1, 0, 0)
            .toInstant(ZoneOffset.UTC),
        ZoneOffset.UTC
    );

    Order order = Order.create(command, fixedClock);

    // When
    Money discount = order.calculateDiscount();

    // Then
    assertThat(discount).isEqualTo(Money.won(1000));
}
```

### 안티패턴 3: 과도한 Mock

**문제**:

```java
❌ BAD: 모든 것을 Mock
@Test
void test() {
    Order order = mock(Order.class);  // Entity를 Mock
    OrderItem item = mock(OrderItem.class);
    Money money = mock(Money.class);

    when(order.getItems()).thenReturn(List.of(item));
    when(item.getSubtotal()).thenReturn(money);
    when(money.getAmount()).thenReturn(BigDecimal.valueOf(10000));

    // 실제로 테스트하는 것이 무엇인지 불명확
}
```

**해결**:

```java
✅ GOOD: 실제 객체 사용
@Test
void test() {
    Order order = OrderFixture.create();  // 실제 객체
    order.addItem(ProductFixture.create(), 2);

    Money total = order.calculateTotal();

    assertThat(total).isEqualTo(Money.won(20000));
}
```

### 안티패턴 4: 테스트가 구현에 종속

**문제**:

```java
❌ BAD: 내부 구현 테스트
@Test
void test() {
    Order order = OrderFixture.create();

    // 내부 메서드 호출 검증
    verify(order).validateStatus();  // 구현 세부사항
    verify(order).calculateTotalAmount();
}
```

**해결**:

```java
✅ GOOD: 동작 테스트
@Test
void test() {
    Order order = OrderFixture.create();

    order.addItem(product, 2);

    // 결과 검증 (what)
    assertThat(order.getItems()).hasSize(1);
    assertThat(order.getTotalAmount()).isEqualTo(Money.won(20000));
}
```

### 안티패턴 5: 너무 긴 테스트

**문제**:

```java
❌ BAD: 100줄 테스트
@Test
void complexTest() {
    // Given 50줄
    // When 30줄
    // Then 20줄
    // 무엇을 테스트하는지 파악 어려움
}
```

**해결**:

```java
✅ GOOD: 작고 집중된 테스트
@Test
void addItem() {
    Order order = OrderFixture.create();

    order.addItem(product, 2);

    assertThat(order.getItems()).hasSize(1);
}

@Test
void calculateTotal() {
    Order order = OrderFixture.createWithItems();

    Money total = order.calculateTotal();

    assertThat(total).isEqualTo(Money.won(20000));
}
```

---

## 테스트 작성 체크리스트

### 단위 테스트

- [ ] 외부 의존성 없이 실행되는가?
- [ ] 100ms 이내에 실행되는가?
- [ ] 테스트 간 독립적인가?
- [ ] 결과가 항상 같은가? (결정적)
- [ ] 실패 시 원인이 명확한가?

### 통합 테스트

- [ ] 실제 외부 시스템과 연동하는가?
- [ ] 트랜잭션이 올바르게 관리되는가?
- [ ] 테스트 데이터가 격리되는가?
- [ ] 1초 이내에 실행되는가?

### 일반

- [ ] Given-When-Then 구조인가?
- [ ] 하나의 개념만 테스트하는가?
- [ ] 테스트 이름이 명확한가?
- [ ] Fixture를 활용하는가?
- [ ] 불필요한 Mock을 사용하지 않는가?

---

이 가이드는 빠르고 안정적이며 유지보수 가능한 테스트 작성을 위한 실용적인 지침을 제공합니다.
