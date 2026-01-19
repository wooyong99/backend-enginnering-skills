# Use Case 작성 가이드

> **목적**: 효과적인 Use Case 설계 및 구현  
> **대상**: 백엔드 개발자 및 AI 개발 어시스턴트  
> **원칙**: Clean Architecture + SOLID

---

## 📋 목차

1. [Use Case 레이어 구조](#use-case-레이어-구조)
2. [Use Case 기본 원칙](#use-case-기본-원칙)
3. [Use Case 구조](#use-case-구조)
4. [Command/Result 설계](#commandresult-설계)
5. [Port 인터페이스 설계](#port-인터페이스-설계)
6. [트랜잭션 관리](#트랜잭션-관리)
7. [예외 처리 전략](#예외-처리-전략)
8. [복잡도 관리](#복잡도-관리)
9. [Use Case 패턴](#use-case-패턴)

---

## Use Case 레이어 구조

### Application Layer 디렉토리 구조

```
application/
└── {도메인명}/
    ├── usecase/
    │   ├── {Action}{Entity}UseCase.java
    │   ├── Create{Entity}UseCase.java
    │   ├── Update{Entity}UseCase.java
    │   ├── Delete{Entity}UseCase.java
    │   └── Get{Entity}UseCase.java
    ├── command/
    │   ├── {Action}{Entity}Command.java
    │   ├── Create{Entity}Command.java
    │   ├── Update{Entity}Command.java
    │   └── Delete{Entity}Command.java
    ├── result/
    │   ├── {Entity}Result.java
    │   └── {Entity}ListResult.java
    ├── port/
    │   ├── in/
    │   │   └── {Action}{Entity}Port.java (선택적)
    │   └── out/
    │       ├── {Entity}Repository.java
    │       ├── {External}Client.java
    │       └── EventPublisher.java
    ├── validator/
    │   └── {Entity}Validator.java
    └── policy/
        └── {Entity}Policy.java
```

### 실제 예시 (주문 도메인)

```
application/
└── order/
    ├── usecase/
    │   ├── CreateOrderUseCase.java
    │   ├── CancelOrderUseCase.java
    │   ├── UpdateOrderUseCase.java
    │   ├── CompleteOrderUseCase.java
    │   └── GetOrderUseCase.java
    ├── command/
    │   ├── CreateOrderCommand.java
    │   ├── CancelOrderCommand.java
    │   ├── UpdateOrderCommand.java
    │   └── CompleteOrderCommand.java
    ├── result/
    │   ├── OrderResult.java
    │   └── OrderListResult.java
    ├── port/
    │   ├── in/
    │   │   └── CreateOrderPort.java (선택적)
    │   └── out/
    │       ├── OrderRepository.java
    │       ├── UserRepository.java
    │       ├── ProductRepository.java
    │       ├── InventoryChecker.java
    │       ├── PaymentProcessor.java
    │       └── OrderEventPublisher.java
    ├── validator/
    │   └── OrderValidator.java
    └── policy/
        ├── PricingPolicy.java
        └── DiscountPolicy.java
```

### 레이어 간 의존성

```
Presentation Layer (Controller)
    ↓ 의존
Application Layer (Use Case)
    ↓ 의존
Domain Layer (Entity, Value Object, Domain Service)
    ↑ 구현
Infrastructure Layer (Repository 구현, External API)
```

**의존성 규칙**:

```
✅ 허용되는 의존성:
- Use Case → Domain Entity
- Use Case → Value Object
- Use Case → Domain Service
- Use Case → Port (인터페이스)
- Infrastructure → Port (구현)

❌ 금지되는 의존성:
- Use Case → Infrastructure (구현체)
- Domain → Use Case
- Domain → Infrastructure
```

### 각 컴포넌트 역할

**1. usecase/**

- 역할: 비즈니스 유스케이스 구현
- 책임: 흐름 조율, 트랜잭션 경계
- 명명: `{Action}{Entity}UseCase.java`

**2. command/**

- 역할: Use Case 입력 데이터
- 책임: 입력 검증, 불변성 보장
- 명명: `{Action}{Entity}Command.java`

**3. result/**

- 역할: Use Case 출력 데이터
- 책임: 결과 데이터 전달, 불변성 보장
- 명명: `{Entity}Result.java`

**4. port/in/** (선택적)

- 역할: Use Case 인터페이스
- 책임: Use Case 진입점 정의
- 명명: `{Action}{Entity}Port.java`
- 사용 시점: Hexagonal Architecture 적용 시

**5. port/out/**

- 역할: 외부 시스템 인터페이스
- 책임: 인프라 추상화
- 명명:
  - Repository: `{Entity}Repository.java`
  - External: `{Service}Client.java`, `{Service}Processor.java`
  - Event: `EventPublisher.java`, `{Entity}EventPublisher.java`

**6. validator/**

- 역할: 복잡한 검증 로직
- 책임: 비즈니스 규칙 검증
- 명명: `{Entity}Validator.java`

**7. policy/**

- 역할: 복잡한 정책 로직
- 책임: 비즈니스 정책 구현
- 명명: `{Policy}Policy.java`

### 파일 배치 예시

**작은 도메인** (파일 수 < 10):

```
application/
└── order/
    ├── CreateOrderUseCase.java
    ├── CreateOrderCommand.java
    ├── OrderResult.java
    └── port/
        ├── OrderRepository.java
        └── PaymentProcessor.java
```

**중간 도메인** (파일 수 10-30):

```
application/
└── order/
    ├── usecase/
    │   ├── CreateOrderUseCase.java
    │   ├── CancelOrderUseCase.java
    │   └── UpdateOrderUseCase.java
    ├── command/
    │   ├── CreateOrderCommand.java
    │   └── CancelOrderCommand.java
    ├── result/
    │   └── OrderResult.java
    └── port/
        ├── OrderRepository.java
        └── PaymentProcessor.java
```

**큰 도메인** (파일 수 > 30):

```
application/
└── order/
    ├── usecase/
    │   ├── CreateOrderUseCase.java
    │   ├── CancelOrderUseCase.java
    │   ├── UpdateOrderUseCase.java
    │   ├── CompleteOrderUseCase.java
    │   └── GetOrderUseCase.java
    ├── command/
    │   ├── CreateOrderCommand.java
    │   ├── CancelOrderCommand.java
    │   ├── UpdateOrderCommand.java
    │   └── CompleteOrderCommand.java
    ├── result/
    │   ├── OrderResult.java
    │   └── OrderListResult.java
    ├── port/
    │   └── out/
    │       ├── OrderRepository.java
    │       ├── PaymentProcessor.java
    │       ├── InventoryChecker.java
    │       └── OrderEventPublisher.java
    ├── validator/
    │   └── OrderValidator.java
    └── policy/
        ├── PricingPolicy.java
        └── DiscountPolicy.java
```

---

## Use Case 기본 원칙

### Use Case 정의

**Use Case**: 사용자의 목표를 달성하기 위한 시스템의 행위

**핵심 특징**:

```
1. 단일 책임
   - 하나의 비즈니스 유스케이스만 처리
   - SRP (Single Responsibility Principle) 준수

2. 흐름 조율
   - 비즈니스 로직 흐름 제어
   - 도메인 객체 조합
   - Port를 통한 외부 시스템 조율

3. 트랜잭션 경계
   - 명확한 트랜잭션 범위
   - 원자성 보장

4. 기술 독립성
   - 특정 프레임워크 의존 최소화
   - 도메인 중심 설계
```

### Use Case vs Service

| 특성   | Use Case                | Traditional Service        |
| ------ | ----------------------- | -------------------------- |
| 관점   | 사용자 목표 중심        | 기술/데이터 중심           |
| 명명   | 동사+명사 (CreateOrder) | 명사Service (OrderService) |
| 책임   | 하나의 유스케이스       | 여러 기능 집합             |
| 메서드 | execute() 하나          | 여러 public 메서드         |
| 크기   | 작고 집중적             | 크고 다목적                |

---

## Use Case 구조

### 기본 템플릿

```
UseCase {
    // 1. 의존성 (Port만)
    private final EntityRepository repository;
    private final ExternalPort externalPort;
    private final EventPublisher eventPublisher;

    // 2. 선택적 의존성
    private final Validator validator;
    private final Policy policy;
    private final DomainService domainService;

    // 3. 진입점 (단일 public 메서드)
    @Transactional
    public Result execute(Command command) {
        // 4. 검증
        validate(command);

        // 5. 비즈니스 로직
        Entity entity = processBusinessLogic(command);

        // 6. 영속화
        repository.save(entity);

        // 7. 이벤트 발행
        publishEvents(entity);

        // 8. 결과 반환
        return Result.from(entity);
    }

    // 9. private 헬퍼 메서드들
    private void validate(Command command) { }
    private Entity processBusinessLogic(Command command) { }
    private void publishEvents(Entity entity) { }
}
```

### 상세 구조 예시

```
CreateOrderUseCase {
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 1. 의존성 선언
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    // Repository (필수)
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;

    // External Port (필요시)
    private final InventoryChecker inventoryChecker;

    // Event Publisher (필요시)
    private final EventPublisher eventPublisher;

    // Validator (복잡한 검증 시)
    private final OrderValidator orderValidator;

    // Policy (복잡한 정책 시)
    private final PricingPolicy pricingPolicy;

    // Domain Service (복잡한 계산 시)
    private final PriceCalculator priceCalculator;

    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 2. 생성자 (의존성 주입)
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    public CreateOrderUseCase(
            OrderRepository orderRepository,
            UserRepository userRepository,
            InventoryChecker inventoryChecker,
            EventPublisher eventPublisher,
            OrderValidator orderValidator,
            PricingPolicy pricingPolicy,
            PriceCalculator priceCalculator) {
        this.orderRepository = orderRepository;
        this.userRepository = userRepository;
        this.inventoryChecker = inventoryChecker;
        this.eventPublisher = eventPublisher;
        this.orderValidator = orderValidator;
        this.pricingPolicy = pricingPolicy;
        this.priceCalculator = priceCalculator;
    }

    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 3. 진입점 메서드 (execute)
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    /**
     * 주문을 생성합니다.
     *
     * @param command 주문 생성 명령
     * @return 생성된 주문 정보
     * @throws UserNotFoundException 사용자를 찾을 수 없는 경우
     * @throws InsufficientStockException 재고 부족
     * @throws InvalidPriceException 가격 계산 오류
     */
    @Transactional
    public OrderResult execute(CreateOrderCommand command) {
        // Phase 1: 검증
        validateCommand(command);

        // Phase 2: 데이터 준비
        User user = loadUser(command.getUserId());
        List<Product> products = loadProducts(command);

        // Phase 3: 비즈니스 로직
        Order order = createOrder(command, user, products);

        // Phase 4: 가격 계산
        calculatePrice(order, user, command);

        // Phase 5: 재고 확인
        checkInventory(order);

        // Phase 6: 영속화
        Order savedOrder = orderRepository.save(order);

        // Phase 7: 이벤트 발행
        publishOrderCreatedEvent(savedOrder);

        // Phase 8: 결과 반환
        return OrderResult.from(savedOrder);
    }

    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    // 4. Private 헬퍼 메서드
    // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    private void validateCommand(CreateOrderCommand command) {
        // 복잡한 검증은 Validator에 위임
        orderValidator.validateForCreate(command);
    }

    private User loadUser(UserId userId) {
        return userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
    }

    private List<Product> loadProducts(CreateOrderCommand command) {
        List<ProductId> productIds = command.getItems().stream()
            .map(OrderItemCommand::getProductId)
            .toList();

        return productRepository.findAllById(productIds);
    }

    private Order createOrder(
            CreateOrderCommand command,
            User user,
            List<Product> products) {

        Order order = Order.create(
            OrderId.generate(),
            user.getId()
        );

        for (OrderItemCommand itemCommand : command.getItems()) {
            Product product = findProduct(products, itemCommand.getProductId());
            order.addItem(product, itemCommand.getQuantity());
        }

        return order;
    }

    private void calculatePrice(
            Order order,
            User user,
            CreateOrderCommand command) {

        // 가격 계산 (Policy 활용)
        Money finalPrice = pricingPolicy.calculateTotalPrice(
            order,
            user.getMemberGrade(),
            command.getCouponIds()
        );

        order.setTotalPrice(finalPrice);
    }

    private void checkInventory(Order order) {
        // 외부 시스템 호출 (Port 활용)
        InventoryCheckResult result =
            inventoryChecker.checkAvailability(order);

        if (!result.isAvailable()) {
            throw new InsufficientStockException(
                result.getUnavailableItems()
            );
        }
    }

    private void publishOrderCreatedEvent(Order order) {
        eventPublisher.publish(
            new OrderCreatedEvent(
                order.getId(),
                order.getUserId(),
                order.getTotalAmount()
            )
        );
    }
}
```

---

## Command/Result 설계

### Command 설계 원칙

**Command 정의**: Use Case의 입력 데이터를 담는 불변 객체

**설계 원칙**:

```
1. 불변성 (Immutability)
   - final 필드
   - Setter 없음
   - Builder 패턴 사용

2. 자가 검증 (Self-Validation)
   - 생성 시점 기본 검증
   - 형식, 필수값 검증

3. 의도 표현 (Intention Revealing)
   - 명확한 명명
   - 도메인 용어 사용

4. 프레임워크 독립성
   - 순수 Java 객체
   - 최소한의 어노테이션
```

### Command 구현 패턴

**기본 패턴**:

```
CreateOrderCommand {
    private final UserId userId;
    private final List<OrderItemCommand> items;
    private final DeliveryAddress deliveryAddress;
    private final List<CouponId> couponIds;

    // private 생성자
    private CreateOrderCommand(
            UserId userId,
            List<OrderItemCommand> items,
            DeliveryAddress deliveryAddress,
            List<CouponId> couponIds) {

        // 기본 검증
        validateNotNull(userId, "사용자 ID는 필수입니다");
        validateNotEmpty(items, "주문 상품은 필수입니다");
        validateNotNull(deliveryAddress, "배송지는 필수입니다");

        this.userId = userId;
        this.items = List.copyOf(items);  // 방어적 복사
        this.deliveryAddress = deliveryAddress;
        this.couponIds = couponIds != null ?
            List.copyOf(couponIds) : List.of();
    }

    // Builder 패턴
    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private UserId userId;
        private List<OrderItemCommand> items = new ArrayList<>();
        private DeliveryAddress deliveryAddress;
        private List<CouponId> couponIds = new ArrayList<>();

        public Builder userId(UserId userId) {
            this.userId = userId;
            return this;
        }

        public Builder items(List<OrderItemCommand> items) {
            this.items = items;
            return this;
        }

        public Builder addItem(OrderItemCommand item) {
            this.items.add(item);
            return this;
        }

        public Builder deliveryAddress(DeliveryAddress address) {
            this.deliveryAddress = address;
            return this;
        }

        public Builder couponIds(List<CouponId> couponIds) {
            this.couponIds = couponIds;
            return this;
        }

        public CreateOrderCommand build() {
            return new CreateOrderCommand(
                userId,
                items,
                deliveryAddress,
                couponIds
            );
        }
    }

    // Getter만 제공
    public UserId getUserId() { return userId; }
    public List<OrderItemCommand> getItems() { return items; }
    public DeliveryAddress getDeliveryAddress() { return deliveryAddress; }
    public List<CouponId> getCouponIds() { return couponIds; }

    // 검증 헬퍼 메서드
    private void validateNotNull(Object value, String message) {
        if (value == null) {
            throw new IllegalArgumentException(message);
        }
    }

    private void validateNotEmpty(Collection<?> value, String message) {
        if (value == null || value.isEmpty()) {
            throw new IllegalArgumentException(message);
        }
    }
}
```

### Result 설계 원칙

**Result 정의**: Use Case의 출력 데이터를 담는 불변 객체

**설계 원칙**:

```
1. 불변성
   - final 필드
   - Setter 없음

2. 필요한 정보만
   - 과도한 정보 지양
   - 클라이언트가 필요한 것만

3. 레이어 독립성
   - 도메인 객체 직접 노출 금지
   - DTO로 변환

4. 명확한 의미
   - 성공/실패 명확히
   - 선택적 정보는 Optional
```

### Result 구현 패턴

**단순 Result**:

```
OrderResult {
    private final OrderId orderId;
    private final Money totalAmount;
    private final OrderStatus status;
    private final LocalDateTime createdAt;

    private OrderResult(
            OrderId orderId,
            Money totalAmount,
            OrderStatus status,
            LocalDateTime createdAt) {
        this.orderId = orderId;
        this.totalAmount = totalAmount;
        this.status = status;
        this.createdAt = createdAt;
    }

    // Factory method (from domain)
    public static OrderResult from(Order order) {
        return new OrderResult(
            order.getId(),
            order.getTotalAmount(),
            order.getStatus(),
            order.getCreatedAt()
        );
    }

    // Getter만 제공
    public OrderId getOrderId() { return orderId; }
    public Money getTotalAmount() { return totalAmount; }
    public OrderStatus getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
}
```

**복합 Result** (여러 정보 포함):

```
PaymentResult {
    private final PaymentId paymentId;
    private final PaymentStatus status;
    private final Money amount;
    private final String transactionId;
    private final LocalDateTime approvedAt;
    private final Optional<String> receiptUrl;

    private PaymentResult(
            PaymentId paymentId,
            PaymentStatus status,
            Money amount,
            String transactionId,
            LocalDateTime approvedAt,
            String receiptUrl) {
        this.paymentId = paymentId;
        this.status = status;
        this.amount = amount;
        this.transactionId = transactionId;
        this.approvedAt = approvedAt;
        this.receiptUrl = Optional.ofNullable(receiptUrl);
    }

    public static PaymentResult success(
            PaymentId paymentId,
            Money amount,
            String transactionId,
            String receiptUrl) {
        return new PaymentResult(
            paymentId,
            PaymentStatus.APPROVED,
            amount,
            transactionId,
            LocalDateTime.now(),
            receiptUrl
        );
    }

    public static PaymentResult failed(
            PaymentId paymentId,
            Money amount) {
        return new PaymentResult(
            paymentId,
            PaymentStatus.FAILED,
            amount,
            null,
            null,
            null
        );
    }
}
```

**Result with validation**:

```
ValidationResult {
    private final boolean valid;
    private final List<String> errors;

    private ValidationResult(boolean valid, List<String> errors) {
        this.valid = valid;
        this.errors = errors != null ? List.copyOf(errors) : List.of();
    }

    public static ValidationResult success() {
        return new ValidationResult(true, List.of());
    }

    public static ValidationResult failure(List<String> errors) {
        return new ValidationResult(false, errors);
    }

    public static ValidationResult failure(String error) {
        return new ValidationResult(false, List.of(error));
    }

    public boolean isValid() {
        return valid;
    }

    public List<String> getErrors() {
        return errors;
    }

    public void throwIfInvalid() {
        if (!valid) {
            throw new ValidationException(errors);
        }
    }
}
```

---

## Port 인터페이스 설계

### Port 설계 원칙

**Port 정의**: Use Case가 외부 시스템과 통신하기 위한 인터페이스

**설계 원칙**:

```
1. 도메인 중심 명명
   - 기술 용어 지양
   - 비즈니스 의미 사용

2. 단일 책임
   - 하나의 관심사만
   - 인터페이스 분리 원칙 (ISP)

3. 명확한 계약
   - JavaDoc 작성
   - 예외 명시
   - 전제조건/후속조건 명시

4. 도메인 타입 사용
   - 원시 타입 지양
   - 값 객체, 엔티티 활용
```

### Repository Port

**기본 패턴**:

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
     * @return 저장된 주문 (ID 포함)
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
    Order findById(OrderId id);

    /**
     * ID로 주문을 조회합니다. (선택적)
     *
     * @param id 주문 ID
     * @return 조회된 주문 또는 empty
     */
    Optional<Order> findByIdOptional(OrderId id);

    /**
     * 사용자의 모든 주문을 조회합니다.
     *
     * @param userId 사용자 ID
     * @return 주문 목록
     */
    List<Order> findByUserId(UserId userId);

    /**
     * 주문 존재 여부를 확인합니다.
     *
     * @param id 주문 ID
     * @return 존재 여부
     */
    boolean existsById(OrderId id);

    /**
     * 주문을 삭제합니다.
     *
     * @param id 주문 ID
     * @throws OrderNotFoundException 주문을 찾을 수 없는 경우
     */
    void delete(OrderId id);
}
```

### External Service Port

**결제 처리 Port**:

```
/**
 * 결제 승인 포트
 *
 * 외부 결제 시스템과 통신합니다.
 */
interface PaymentProcessor {

    /**
     * 결제를 승인합니다.
     *
     * @param request 결제 요청 정보
     * @return 결제 승인 결과
     * @throws PaymentDeclinedException 결제 거부
     * @throws PaymentServiceUnavailableException 결제 서비스 장애
     * @throws PaymentTimeoutException 결제 서비스 응답 시간 초과
     */
    PaymentApprovalResult approve(PaymentRequest request);

    /**
     * 결제 상태를 조회합니다.
     *
     * @param transactionId 거래 ID
     * @return 결제 상태
     * @throws TransactionNotFoundException 거래를 찾을 수 없는 경우
     */
    PaymentStatus getStatus(String transactionId);

    /**
     * 결제를 취소합니다.
     *
     * @param transactionId 거래 ID
     * @param amount 취소 금액
     * @return 취소 결과
     * @throws CancellationNotAllowedException 취소 불가 상태
     */
    CancellationResult cancel(String transactionId, Money amount);
}
```

**알림 Port**:

```
/**
 * 알림 발송 포트
 */
interface NotificationSender {

    /**
     * 이메일을 발송합니다.
     *
     * @param recipient 수신자
     * @param subject 제목
     * @param content 내용
     * @throws EmailSendException 이메일 발송 실패
     */
    void sendEmail(Email recipient, String subject, String content);

    /**
     * SMS를 발송합니다.
     *
     * @param recipient 수신자
     * @param message 메시지
     * @throws SmsSendException SMS 발송 실패
     */
    void sendSms(PhoneNumber recipient, String message);

    /**
     * 푸시 알림을 발송합니다.
     *
     * @param userId 사용자 ID
     * @param title 제목
     * @param body 내용
     * @throws PushSendException 푸시 발송 실패
     */
    void sendPush(UserId userId, String title, String body);
}
```

### Event Publisher Port

```
/**
 * 이벤트 발행 포트
 *
 * 도메인 이벤트를 발행합니다.
 */
interface EventPublisher {

    /**
     * 이벤트를 발행합니다.
     *
     * @param event 발행할 이벤트
     * @param <T> 이벤트 타입
     */
    <T extends DomainEvent> void publish(T event);

    /**
     * 여러 이벤트를 일괄 발행합니다.
     *
     * @param events 발행할 이벤트 목록
     */
    void publishAll(List<? extends DomainEvent> events);
}
```

---

## 트랜잭션 관리

### 트랜잭션 원칙

```
1. Use Case = 트랜잭션 경계
   - 하나의 Use Case가 하나의 트랜잭션
   - @Transactional은 Use Case에만

2. 하나의 Aggregate만 수정
   - 여러 Aggregate 수정 금지
   - 이벤트로 분리

3. 읽기 전용 최적화
   - 조회는 @Transactional(readOnly = true)
   - 불필요한 더티 체킹 방지
```

### 트랜잭션 패턴

**쓰기 트랜잭션**:

```
CreateOrderUseCase {
    @Transactional
    public OrderResult execute(CreateOrderCommand command) {
        // 1. 엔티티 생성
        Order order = Order.create(command);

        // 2. 비즈니스 로직
        order.place();

        // 3. 저장 (트랜잭션 커밋 시 반영)
        orderRepository.save(order);

        // 4. 이벤트 발행 (트랜잭션 커밋 후)
        eventPublisher.publish(new OrderCreatedEvent(order.getId()));

        return OrderResult.from(order);
    }
}
```

**읽기 전용 트랜잭션**:

```
GetOrderUseCase {
    @Transactional(readOnly = true)
    public OrderResult execute(GetOrderQuery query) {
        Order order = orderRepository.findById(query.getOrderId());
        return OrderResult.from(order);
    }
}
```

**중첩 트랜잭션 회피**:

```
// ❌ 잘못된 예: Use Case가 다른 Use Case 호출
CreateOrderUseCase {
    @Transactional
    public OrderResult execute(CreateOrderCommand command) {
        Order order = createOrder(command);

        // ❌ 다른 Use Case 호출 (중첩 트랜잭션)
        processPaymentUseCase.execute(...);

        return OrderResult.from(order);
    }
}

// ✅ 올바른 예: 이벤트로 분리
CreateOrderUseCase {
    @Transactional
    public OrderResult execute(CreateOrderCommand command) {
        Order order = createOrder(command);
        orderRepository.save(order);

        // ✅ 이벤트 발행
        eventPublisher.publish(new OrderCreatedEvent(order.getId()));

        return OrderResult.from(order);
    }
}

PaymentEventHandler {
    @EventListener
    @Transactional
    public void handle(OrderCreatedEvent event) {
        // 별도 트랜잭션
        processPaymentUseCase.execute(...);
    }
}
```

---

## 예외 처리 전략

### 예외 계층

```
Exception
  └─ RuntimeException
      └─ DomainException (Base)
          ├─ ValidationException
          │   ├─ InvalidCommandException
          │   └─ InvalidStateException
          ├─ BusinessRuleViolationException
          │   ├─ InsufficientStockException
          │   └─ PaymentDeclinedException
          ├─ ResourceNotFoundException
          │   ├─ OrderNotFoundException
          │   └─ UserNotFoundException
          └─ ExternalServiceException
              ├─ PaymentServiceException
              └─ NotificationServiceException
```

### 예외 처리 패턴

**Use Case에서의 예외 처리**:

```
CreateOrderUseCase {
    public OrderResult execute(CreateOrderCommand command) {
        try {
            // 1. 검증 (예외 발생 가능)
            validator.validateForCreate(command);

            // 2. 비즈니스 로직 (예외 발생 가능)
            Order order = createOrder(command);

            // 3. 외부 시스템 호출 (예외 처리)
            checkInventory(order);

            // 4. 저장
            orderRepository.save(order);

            return OrderResult.from(order);

        } catch (ValidationException e) {
            // 검증 실패: 그대로 전파
            throw e;

        } catch (BusinessRuleViolationException e) {
            // 비즈니스 규칙 위반: 그대로 전파
            throw e;

        } catch (ExternalServiceException e) {
            // 외부 서비스 오류: 로깅 후 전파
            logger.error("External service error", e);
            throw new OrderCreationFailedException(
                "주문 생성 중 오류가 발생했습니다", e
            );
        }
    }

    private void checkInventory(Order order) {
        try {
            inventoryChecker.checkAvailability(order);
        } catch (ExternalApiException e) {
            // 외부 API 예외를 도메인 예외로 변환
            throw new InventoryCheckException(
                "재고 확인 중 오류가 발생했습니다", e
            );
        }
    }
}
```

**예외 메시지 국제화**:

```
DomainException {
    private final String errorCode;
    private final Object[] args;

    protected DomainException(
            String errorCode,
            Object... args) {
        super(formatMessage(errorCode, args));
        this.errorCode = errorCode;
        this.args = args;
    }

    public String getErrorCode() {
        return errorCode;
    }

    public Object[] getArgs() {
        return args;
    }
}

OrderNotFoundException extends ResourceNotFoundException {
    public OrderNotFoundException(OrderId orderId) {
        super("order.not.found", orderId.getValue());
    }
}

// messages.properties
// order.not.found=주문을 찾을 수 없습니다: {0}
```

---

## 복잡도 관리

### 복잡도 측정 기준

```
✅ 적절한 Use Case (30-50줄):
- 명확한 흐름
- 읽기 쉬움
- 테스트 용이

⚠️  복잡한 Use Case (50-100줄):
- 리팩토링 고려
- Validator/Policy 추출 검토

❌ 과도하게 복잡한 Use Case (100줄 이상):
- 즉시 리팩토링 필요
- 책임 분리
```

### 복잡도 감소 패턴

**패턴 1: Validator 추출**:

```
// Before (복잡함)
CreateOrderUseCase {
    public OrderResult execute(CreateOrderCommand command) {
        // 검증 로직 30줄
        User user = userRepository.findById(command.getUserId())
            .orElseThrow(() -> new UserNotFoundException());
        if (user.isDeleted()) throw new DeletedUserException();
        if (!user.isEmailVerified()) throw new EmailNotVerifiedException();

        for (OrderItemCommand item : command.getItems()) {
            Product product = productRepository.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException());
            if (!product.isAvailable()) throw new ProductNotAvailableException();
            if (product.getStock() < item.getQuantity())
                throw new InsufficientStockException();
        }
        // ... 비즈니스 로직
    }
}

// After (간결함)
CreateOrderUseCase {
    private final OrderValidator validator;

    public OrderResult execute(CreateOrderCommand command) {
        // 검증 위임 (1줄)
        validator.validateForCreate(command);

        // 비즈니스 로직에 집중
        Order order = createOrder(command);
        orderRepository.save(order);
        return OrderResult.from(order);
    }
}

OrderValidator {
    public void validateForCreate(CreateOrderCommand command) {
        validateUser(command.getUserId());
        validateProducts(command.getItems());
    }

    private void validateUser(UserId userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException());
        if (user.isDeleted()) throw new DeletedUserException();
        if (!user.isEmailVerified()) throw new EmailNotVerifiedException();
    }

    private void validateProducts(List<OrderItemCommand> items) {
        for (OrderItemCommand item : items) {
            validateProduct(item);
        }
    }
}
```

**패턴 2: Policy 추출**:

```
// Before (복잡함)
CreateOrderUseCase {
    public OrderResult execute(CreateOrderCommand command) {
        Order order = createOrder(command);

        // 할인 계산 로직 40줄
        Money discount = Money.ZERO;

        if (user.getMemberGrade() == MemberGrade.VIP) {
            discount = order.getTotalAmount().multiply(0.15);
        } else if (user.getMemberGrade() == MemberGrade.GOLD) {
            discount = order.getTotalAmount().multiply(0.10);
        }

        for (Coupon coupon : coupons) {
            if (coupon.isPercentage()) {
                discount = discount.add(
                    order.getTotalAmount().multiply(coupon.getRate())
                );
            } else {
                discount = discount.add(coupon.getAmount());
            }
        }

        // ...
    }
}

// After (간결함)
CreateOrderUseCase {
    private final DiscountPolicy discountPolicy;

    public OrderResult execute(CreateOrderCommand command) {
        Order order = createOrder(command);

        // 할인 계산 위임 (1줄)
        Money discount = discountPolicy.calculateDiscount(
            order,
            user.getMemberGrade(),
            coupons
        );

        order.applyDiscount(discount);
        orderRepository.save(order);
        return OrderResult.from(order);
    }
}
```

**패턴 3: Helper 메서드 추출**:

```
CreateOrderUseCase {
    public OrderResult execute(CreateOrderCommand command) {
        // Phase 1
        User user = loadUser(command);
        List<Product> products = loadProducts(command);

        // Phase 2
        Order order = createOrder(command, user, products);

        // Phase 3
        applyDiscounts(order, user, command);

        // Phase 4
        orderRepository.save(order);
        publishEvents(order);

        return OrderResult.from(order);
    }

    private User loadUser(CreateOrderCommand command) {
        return userRepository.findById(command.getUserId())
            .orElseThrow(() -> new UserNotFoundException());
    }

    private List<Product> loadProducts(CreateOrderCommand command) {
        // ...
    }

    private Order createOrder(
            CreateOrderCommand command,
            User user,
            List<Product> products) {
        // ...
    }

    private void applyDiscounts(
            Order order,
            User user,
            CreateOrderCommand command) {
        // ...
    }

    private void publishEvents(Order order) {
        // ...
    }
}
```

---

## Use Case 패턴

### 패턴 1: 생성 (Create)

```
CreateEntityUseCase {
    private final EntityRepository repository;
    private final EventPublisher eventPublisher;

    @Transactional
    public EntityResult execute(CreateEntityCommand command) {
        // 1. 중복 검사
        validateUniqueness(command);

        // 2. 엔티티 생성
        Entity entity = Entity.create(command);

        // 3. 저장
        Entity saved = repository.save(entity);

        // 4. 이벤트 발행
        eventPublisher.publish(new EntityCreatedEvent(saved.getId()));

        // 5. 결과 반환
        return EntityResult.from(saved);
    }

    private void validateUniqueness(CreateEntityCommand command) {
        if (repository.existsByName(command.getName())) {
            throw new DuplicateEntityException(command.getName());
        }
    }
}
```

### 패턴 2: 수정 (Update)

```
UpdateEntityUseCase {
    private final EntityRepository repository;

    @Transactional
    public EntityResult execute(UpdateEntityCommand command) {
        // 1. 엔티티 조회
        Entity entity = repository.findById(command.getId());

        // 2. 수정 (도메인 메서드 호출)
        entity.update(
            command.getName(),
            command.getDescription()
        );

        // 3. 저장 (더티 체킹)
        // repository.save(entity); // 일반적으로 불필요

        // 4. 결과 반환
        return EntityResult.from(entity);
    }
}
```

### 패턴 3: 삭제 (Delete)

```
DeleteEntityUseCase {
    private final EntityRepository repository;
    private final EventPublisher eventPublisher;

    @Transactional
    public void execute(DeleteEntityCommand command) {
        // 1. 엔티티 조회
        Entity entity = repository.findById(command.getId());

        // 2. 삭제 가능 여부 검증
        entity.validateCanDelete();

        // 3. 소프트 삭제 (권장)
        entity.delete();

        // 또는 하드 삭제
        // repository.delete(entity.getId());

        // 4. 이벤트 발행
        eventPublisher.publish(new EntityDeletedEvent(entity.getId()));
    }
}
```

### 패턴 4: 상태 전이 (State Transition)

```
ApproveEntityUseCase {
    private final EntityRepository repository;
    private final NotificationSender notificationSender;
    private final EventPublisher eventPublisher;

    @Transactional
    public EntityResult execute(ApproveEntityCommand command) {
        // 1. 엔티티 조회
        Entity entity = repository.findById(command.getId());

        // 2. 상태 전이 (도메인 메서드)
        entity.approve(command.getApprover());

        // 3. 알림
        notificationSender.sendApprovalNotification(entity);

        // 4. 이벤트 발행
        eventPublisher.publish(new EntityApprovedEvent(entity.getId()));

        // 5. 결과 반환
        return EntityResult.from(entity);
    }
}
```

### 패턴 5: 배치 처리 (Batch)

```
BulkUpdateEntityUseCase {
    private final EntityRepository repository;

    @Transactional
    public BulkUpdateResult execute(BulkUpdateCommand command) {
        List<EntityId> successIds = new ArrayList<>();
        List<EntityId> failureIds = new ArrayList<>();

        for (EntityId id : command.getIds()) {
            try {
                Entity entity = repository.findById(id);
                entity.update(command.getUpdateData());
                successIds.add(id);
            } catch (Exception e) {
                failureIds.add(id);
                logger.error("Failed to update entity: {}", id, e);
            }
        }

        return new BulkUpdateResult(successIds, failureIds);
    }
}
```

---

이 가이드는 효과적인 Use Case 작성을 위한 실용적인 패턴과 원칙을 제공합니다. 프로젝트의 요구사항에 맞게 조정하여 사용하시기 바랍니다.
