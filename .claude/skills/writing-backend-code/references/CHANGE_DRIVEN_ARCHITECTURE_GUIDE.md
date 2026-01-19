# 변경 이유 기반 아키텍처 설계 가이드

> **목적**: 변경 비용 최소화와 응집도 극대화를 위한 설계 판단 기준  
> **대상**: 백엔드 개발자 및 AI 개발 어시스턴트  
> **원칙**: Reason to Change > Feature | Technology

---

## 📋 목차

1. [핵심 설계 철학](#핵심-설계-철학)
2. [Domain vs Application 레이어의 본질](#domain-vs-application-레이어의-본질)
3. [변경 이유 분석 프레임워크](#변경-이유-분석-프레임워크)
4. [Bounded Context 내부 모듈 분리](#bounded-context-내부-모듈-분리)
5. [Domain-Application 매핑 패턴](#domain-application-매핑-패턴)
6. [실전 설계 판단 사례](#실전-설계-판단-사례)
7. [리팩토링 트리거](#리팩토링-트리거)

---

## 핵심 설계 철학

### 변경 이유가 구조를 결정한다

**잘못된 질문**:

```
❌ "이 기능은 어느 레이어에 넣어야 하나?"
❌ "REST API니까 Controller를 만들어야 하나?"
❌ "User 도메인이니까 user 패키지에 넣어야 하나?"
```

**올바른 질문**:

```
✅ "이 코드는 왜 변경되는가?"
✅ "무엇이 변경될 때 이 코드가 함께 변경되는가?"
✅ "이 변경이 다른 무엇에 영향을 주는가?"
```

### 설계의 본질적 목표

```
목표 1: 변경의 격리
→ A가 변경될 때 B는 변경되지 않아야 함

목표 2: 변경의 응집
→ 같은 이유로 변경되는 것들은 함께 있어야 함

목표 3: 변경의 예측 가능성
→ 변경의 영향 범위를 쉽게 파악할 수 있어야 함
```

### 변경 이유의 계층

```
Level 1: 비즈니스 정책 변경
예: "할인율 계산 방식 변경", "주문 취소 규칙 변경"
→ Domain Layer

Level 2: 비즈니스 플로우 변경
예: "주문 생성 시 재고 확인 순서 변경", "알림 발송 시점 변경"
→ Application Layer

Level 3: 외부 시스템 변경
예: "결제 PG사 변경", "메시지 큐 변경"
→ Infrastructure Layer

Level 4: 프레임워크/기술 변경
예: "Spring → Quarkus", "PostgreSQL → MongoDB"
→ Infrastructure Layer

Level 5: UI/프로토콜 변경
예: "REST → GraphQL", "Web → Mobile"
→ Presentation Layer
```

---

## Domain vs Application 레이어의 본질

### 레이어를 나누는 진짜 이유

**기능으로 나누지 않는다. 변경 이유로 나눈다.**

```
❌ 잘못된 이해:
"Domain은 비즈니스 로직, Application은 서비스 로직"
→ 모호함. 둘 다 비즈니스 로직일 수 있음

✅ 올바른 이해:
"Domain은 비즈니스 본질, Application은 비즈니스 사용 방식"
→ 변경 이유가 다름
```

### Domain Layer: "무엇(What)"의 레이어

**변경 이유**: 비즈니스 규칙 자체가 변경될 때

**질문**:

- "이 비즈니스는 본질적으로 무엇인가?"
- "이 개념은 어떤 시스템에서도 동일한가?"
- "UI가 바뀌어도, 사용 방식이 바뀌어도 이 규칙은 동일한가?"

**예시**:

```java
// ✅ Domain Layer
// 변경 이유: "주문의 본질적인 규칙이 바뀔 때"

class Order {
    // "주문은 최소 1개 이상의 상품을 포함해야 한다"
    // → 이것은 주문의 본질. 어떤 시스템에서든 동일
    void addItem(OrderItem item) {
        if (item == null) {
            throw new IllegalArgumentException("상품은 필수입니다");
        }
        this.items.add(item);
    }

    // "배송 시작된 주문은 취소할 수 없다"
    // → 이것도 주문의 본질적 규칙
    void cancel() {
        if (this.status == OrderStatus.SHIPPED) {
            throw new OrderAlreadyShippedException();
        }
        this.status = OrderStatus.CANCELLED;
    }

    // "주문 금액은 상품 가격의 합이다"
    // → 주문 금액 계산의 본질
    Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

**이런 코드는 Domain이 아니다**:

```java
// ❌ Application 관심사
void createOrderAndSendEmail() { }  // "그리고"가 있으면 의심
void validateAndSave() { }          // 흐름 제어
void processWithRetry() { }         // 기술적 처리
```

### Application Layer: "어떻게(How)"의 레이어

**변경 이유**: 비즈니스 사용 방식이 변경될 때

**질문**:

- "이 유스케이스는 어떤 순서로 실행되는가?"
- "어떤 외부 시스템과 협력하는가?"
- "실패하면 어떻게 처리하는가?"

**예시**:

```java
// ✅ Application Layer
// 변경 이유: "주문 생성 프로세스가 바뀔 때"

class CreateOrderUseCase {

    // 이것은 "주문 생성"이라는 유스케이스의 실행 방식
    // 변경 이유:
    // - "재고 확인을 먼저 할지, 결제를 먼저 할지" 순서 변경
    // - "알림을 즉시 보낼지, 비동기로 보낼지" 방식 변경
    // - "재고가 없으면 예약 주문으로 할지, 실패 처리할지" 정책 변경

    @Transactional
    public OrderResult execute(CreateOrderCommand command) {
        // 1. 재고 확인 (외부 시스템)
        checkInventory(command);

        // 2. 주문 생성 (도메인 규칙)
        Order order = Order.create(command);

        // 3. 결제 처리 (외부 시스템)
        processPayment(order);

        // 4. 저장
        orderRepository.save(order);

        // 5. 이벤트 발행
        eventPublisher.publish(new OrderCreatedEvent(order));

        return OrderResult.from(order);
    }
}
```

### 핵심 차이점

| 관점          | Domain Layer             | Application Layer         |
| ------------- | ------------------------ | ------------------------- |
| **변경 이유** | 비즈니스 규칙 자체       | 비즈니스 사용 방식        |
| **질문**      | "무엇이 올바른가?"       | "어떻게 사용하는가?"      |
| **관심사**    | 불변식, 규칙, 계산       | 흐름, 조율, 트랜잭션      |
| **의존성**    | 없음 (순수)              | 많음 (Port, Repository)   |
| **재사용성**  | 높음 (여러 Use Case에서) | 낮음 (특정 Use Case 전용) |
| **테스트**    | 단위 테스트 (빠름)       | 통합 테스트 (느림)        |

**실전 판단 예시**:

```java
// 질문: 이 코드는 어디에?
money.multiply(0.1)

답변:
✅ Domain - Money 클래스의 메서드로
   이유: "금액 곱셈"은 금액의 본질적 연산. 어떤 유스케이스든 동일

// 질문: 이 코드는 어디에?
if (user.isVIP()) { return price.multiply(0.9); }

답변:
✅ Domain - DiscountPolicy 또는 Order 클래스로
   이유: "VIP는 10% 할인"은 비즈니스 규칙. 본질적 정책

// 질문: 이 코드는 어디에?
Order order = createOrder();
sendEmail(order);
updateInventory(order);

답변:
✅ Application - Use Case로
   이유: "주문 생성 후 이메일을 보내고 재고를 업데이트"는 실행 방식.
        순서를 바꿀 수 있고, 이메일을 안 보낼 수도 있음
```

---

## 변경 이유 분석 프레임워크

### STEP 1: 변경 시나리오 식별

**변경 시나리오 수집 방법**:

```
1. 과거 변경 이력 분석
"지난 6개월간 이 코드는 왜 변경되었나?"

2. 미래 변경 예측
"앞으로 이것은 왜 변경될 것 같은가?"

3. 이해관계자 인터뷰
"이 비즈니스는 어떻게 변할 수 있나?"
```

**예시: 주문 도메인**

```
변경 시나리오 1: "할인 정책 변경"
- VIP 15% → 20%
- 쿠폰 중복 사용 허용
→ 영향: 가격 계산 로직

변경 시나리오 2: "주문 생성 프로세스 변경"
- 재고 확인 순서 변경
- 결제 실패 시 재시도 로직 추가
→ 영향: Use Case 흐름

변경 시나리오 3: "결제 PG사 변경"
- 토스페이먼츠 → 포트원
→ 영향: Infrastructure

변경 시나리오 4: "주문 취소 규칙 변경"
- 배송 준비 중에도 취소 가능
→ 영향: Order 엔티티
```

### STEP 2: 변경의 성격 분류

**분류 기준**:

```
Type A: 본질 변경 (What)
- 비즈니스 규칙이 바뀜
- 예: "VIP 할인율 15% → 20%"
→ Domain Layer

Type B: 방식 변경 (How)
- 실행 순서/방법이 바뀜
- 예: "재고 확인을 먼저 한다"
→ Application Layer

Type C: 기술 변경 (Tech)
- 구현 기술이 바뀜
- 예: "결제 PG사를 바꾼다"
→ Infrastructure Layer

Type D: 인터페이스 변경 (UI)
- 사용자 접점이 바뀜
- 예: "REST API → GraphQL"
→ Presentation Layer
```

### STEP 3: 영향도 매핑

**변경 영향도 분석**:

```
변경: "VIP 할인율 15% → 20%"

현재 구조:
CreateOrderUseCase {
    Order order = Order.create(command);
    if (user.isVIP()) {
        order.applyDiscount(0.15);  // ← 여기 변경
    }
}

영향 범위: Use Case, 테스트 코드
문제: 같은 규칙이 여러 Use Case에 중복

개선된 구조:
DiscountPolicy {
    Money calculate(Order order, User user) {
        if (user.isVIP()) {
            return order.getTotal().multiply(0.20);  // ← 여기만 변경
        }
    }
}

영향 범위: DiscountPolicy 클래스만
효과: 변경 격리, 중복 제거
```

### STEP 4: 응집도 검증

**같은 이유로 변경되는가?**

```
❌ 낮은 응집도:
OrderService {
    void createOrder() { }
    void sendEmail() { }
    void updateInventory() { }
}

변경 이유:
- 주문 생성 규칙 변경 → createOrder 수정
- 이메일 템플릿 변경 → sendEmail 수정
- 재고 관리 방식 변경 → updateInventory 수정

→ 서로 다른 이유로 변경됨 → 낮은 응집도

✅ 높은 응집도:
CreateOrderUseCase {
    void execute() {
        createOrder();
        reserveInventory();
        publishEvent();
    }
}

변경 이유:
- "주문 생성 프로세스" 변경
  → 전체 흐름이 함께 변경됨

→ 같은 이유로 변경됨 → 높은 응집도
```

---

## Bounded Context 내부 모듈 분리

### 모듈 분리의 진짜 질문

**잘못된 질문**:

```
❌ "User 관련 기능이니까 user 모듈?"
❌ "테이블이 다르니까 모듈을 나눈다?"
```

**올바른 질문**:

```
✅ "이것들이 같은 이유로 변경되는가?"
✅ "이것들이 독립적으로 변경될 수 있는가?"
✅ "이것들을 분리하면 변경 비용이 줄어드는가?"
```

### 변경 이유 기반 모듈 분리

**사례 1: 주문 컨텍스트**

```
# 잘못된 분리 (기능 기준)
order/
├── order/          # 주문
├── payment/        # 결제
├── delivery/       # 배송
└── cancel/         # 취소

문제:
"주문 생성 규칙 변경" → order, payment, delivery 모두 변경
→ 변경이 분산됨

# 올바른 분리 (변경 이유 기준)
order/
├── order-lifecycle/      # 주문 생명주기
│   ├── Order
│   ├── OrderStatus
│   ├── CreateOrderUseCase
│   ├── CancelOrderUseCase
│   └── CompleteOrderUseCase
│
├── order-pricing/        # 가격 정책
│   ├── PricingPolicy
│   ├── DiscountPolicy
│   └── PriceCalculator
│
└── order-fulfillment/   # 주문 이행
    ├── Inventory
    ├── Delivery
    └── DeliveryTracker

변경 시나리오별 영향:
"할인 정책 변경" → order-pricing만 변경
"배송 프로세스 변경" → order-fulfillment만 변경
"주문 상태 전이 변경" → order-lifecycle만 변경
```

**사례 2: 사용자 컨텍스트**

```
# 잘못된 분리
user/
└── User  # 모든 사용자 관련 기능

문제:
"로그인 방식 변경"과 "포인트 적립 규칙 변경"이 같은 파일 수정
→ 서로 다른 이유의 변경이 섞임

# 올바른 분리
user/
├── user-identity/        # 신원 관리
│   ├── User
│   ├── Authentication
│   └── LoginUseCase
│
├── user-profile/         # 프로필 관리
│   ├── UserProfile
│   └── UpdateProfileUseCase
│
└── user-reward/         # 리워드
    ├── Point
    ├── PointPolicy
    └── EarnPointUseCase

변경 시나리오별 영향:
"로그인 방식 변경" → user-identity만 변경
"포인트 규칙 변경" → user-reward만 변경
"프로필 항목 추가" → user-profile만 변경
```

### 분리 판단 결정 트리

```
┌─────────────────────────────────┐
│ 두 개념 A, B가 있다             │
└────────────┬────────────────────┘
             ▼
┌─────────────────────────────────┐
│ A와 B가 같은 이유로 변경되는가? │
└────┬─────────────────────┬──────┘
     │ Yes                 │ No
     ▼                     ▼
┌─────────────┐      ┌──────────────┐
│ 같은 모듈   │      │ 다른 모듈    │
└─────────────┘      └──────┬───────┘
                            ▼
                     ┌──────────────────────┐
                     │ A 변경 시 B도         │
                     │ 변경되는가?           │
                     └──┬────────────┬──────┘
                        │ Yes        │ No
                        ▼            ▼
                  ┌──────────┐  ┌──────────┐
                  │ 의존성   │  │ 완전 분리│
                  │ 검토     │  │          │
                  └──────────┘  └──────────┘
```

**예시 적용**:

```
질문: Order와 Payment는 같은 모듈인가?

1. 같은 이유로 변경되는가?
   - "주문 취소 규칙 변경" → Order만 변경
   - "결제 PG사 변경" → Payment만 변경
   → No (다른 이유)

2. Order 변경 시 Payment도 변경되는가?
   - "주문 금액 계산 방식 변경" → Order, Payment 모두 변경 가능
   → Yes (의존성 있음)

3. 결론:
   - 같은 모듈 X
   - 하지만 Payment가 Order에 의존
   - Payment → Order (단방향 의존성만 허용)
```

### 모듈 응집도 측정

**높은 응집도의 신호**:

```
✅ 하나의 변경 시나리오가 하나의 모듈만 영향
✅ 모듈 내부의 클래스들이 함께 변경됨
✅ 모듈의 public 인터페이스가 안정적
```

**낮은 응집도의 신호**:

```
❌ 하나의 변경이 여러 모듈에 분산
❌ 모듈 내부에서 일부만 변경됨
❌ 모듈의 public 인터페이스가 자주 변경됨
```

---

## Domain-Application 매핑 패턴

### 매핑 패턴의 본질

**핵심 원칙**:

```
Domain과 Application은 1:1이 아니다.
왜냐하면 변경 이유가 다르기 때문이다.

Domain: "무엇이 올바른가?" (비즈니스 규칙)
Application: "어떻게 사용하는가?" (유스케이스)

→ 하나의 비즈니스 규칙은 여러 방식으로 사용될 수 있다
→ 하나의 유스케이스는 여러 비즈니스 규칙을 사용할 수 있다
```

### 패턴 1: 1 Domain : N Application

**언제**: 하나의 도메인을 여러 방식으로 사용할 때

**변경 이유 분석**:

```
Domain 변경 이유: 비즈니스 규칙 변경
Application 변경 이유: 사용 방식 변경

→ 변경 이유가 다르므로 분리
```

**예시 1: Order 도메인**

```
Domain Layer:
order/
└── domain/
    ├── Order.java              # 주문 엔티티
    ├── OrderItem.java
    ├── OrderStatus.java
    └── PricingPolicy.java      # 가격 정책

Application Layer:
order/
└── application/
    ├── CreateOrderUseCase.java      # 일반 주문 생성
    ├── CreateQuickOrderUseCase.java # 원클릭 주문 생성
    ├── CreateSubscriptionOrderUseCase.java  # 구독 주문 생성
    ├── CancelOrderUseCase.java
    └── UpdateOrderUseCase.java

변경 시나리오:
"할인율 계산 방식 변경" → PricingPolicy만 변경
                        → 모든 Use Case는 자동으로 반영

"원클릭 주문 프로세스 변경" → CreateQuickOrderUseCase만 변경
                           → 다른 Use Case는 영향 없음
```

**왜 이렇게 설계하는가?**

```
비즈니스 규칙은 하나 (Domain):
- "주문 금액은 상품 가격의 합이다"
- "VIP는 15% 할인한다"

사용 방식은 여러 개 (Application):
- 일반 주문: 장바구니 → 주문서 → 결제
- 원클릭 주문: 바로 결제
- 구독 주문: 정기 결제

→ 같은 규칙을 다른 방식으로 사용
→ 1 Domain : N Application
```

**예시 2: User 도메인**

```
Domain Layer:
user/
└── domain/
    ├── User.java
    ├── Password.java
    └── PasswordPolicy.java     # 비밀번호 정책

Application Layer:
user/
└── application/
    ├── RegisterUserUseCase.java         # 회원가입
    ├── LoginUserUseCase.java            # 로그인
    ├── ChangePasswordUseCase.java       # 비밀번호 변경
    ├── ResetPasswordUseCase.java        # 비밀번호 재설정
    └── UpdateProfileUseCase.java

변경 시나리오:
"비밀번호 정책 변경" → PasswordPolicy만 변경
                     → 모든 비밀번호 관련 Use Case 자동 반영

"로그인 프로세스 변경" → LoginUserUseCase만 변경
"회원가입 프로세스 변경" → RegisterUserUseCase만 변경
```

### 패턴 2: 0 Domain : 1 Application

**언제**: 도메인 규칙 없이 조율만 하는 Use Case

**변경 이유 분석**:

```
이 Use Case는:
- 비즈니스 규칙을 구현하지 않음
- 기존 도메인들을 조율만 함
- 프로세스/흐름이 핵심

→ Application만 존재 (Domain 없음)
```

**예시 1: 주문 완료 알림**

```
Application Layer:
notification/
└── application/
    └── SendOrderCompletionNotificationUseCase.java

@UseCase
class SendOrderCompletionNotificationUseCase {
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final EmailSender emailSender;
    private final SmsSender smsSender;

    public void execute(SendNotificationCommand command) {
        // 데이터 조회
        Order order = orderRepository.findById(command.getOrderId());
        User user = userRepository.findById(order.getUserId());

        // 알림 발송 (조율만 함)
        emailSender.send(user.getEmail(), createEmailContent(order));
        smsSender.send(user.getPhone(), createSmsContent(order));
    }
}

Domain Layer:
(없음)

왜?
- "알림을 보낸다"는 비즈니스 규칙이 아님
- 단지 "주문 완료 시 이메일과 SMS를 보낸다"는 프로세스
- 도메인 규칙(Order, User)은 이미 존재
- 이 Use Case는 조율만 함

변경 이유:
"알림 발송 순서 변경" → Use Case만 변경
"알림 채널 추가 (푸시)" → Use Case만 변경
```

**예시 2: 데이터 동기화**

```
Application Layer:
sync/
└── application/
    └── SyncOrderToDataWarehouseUseCase.java

@UseCase
class SyncOrderToDataWarehouseUseCase {
    private final OrderRepository orderRepository;
    private final DataWarehouseClient dataWarehouseClient;

    public void execute(SyncCommand command) {
        // 조회
        List<Order> orders = orderRepository.findModifiedAfter(
            command.getLastSyncTime()
        );

        // 변환
        List<OrderData> dataList = orders.stream()
            .map(this::toDataWarehouseFormat)
            .toList();

        // 전송
        dataWarehouseClient.bulkInsert(dataList);
    }
}

Domain Layer:
(없음)

왜?
- "데이터를 동기화한다"는 기술적 프로세스
- 비즈니스 규칙 없음
- 기존 도메인(Order)을 읽어서 전송만 함

변경 이유:
"동기화 주기 변경" → Use Case만 변경
"동기화 대상 변경" → Use Case만 변경
```

**언제 Domain을 만들지 말아야 하는가?**

```
✅ Domain 없이 Application만:
- 단순 CRUD
- 데이터 조회 및 변환
- 외부 시스템 호출 조율
- 알림/리포팅 등 부가 기능

❌ Domain이 필요한 경우:
- 비즈니스 규칙이 있음
- 상태 전이가 복잡함
- 불변식을 지켜야 함
- 계산 로직이 있음
```

### 패턴 3: N Domain : 1 Application

**언제**: 여러 도메인을 협력시켜야 하는 Use Case

**변경 이유 분석**:

```
각 Domain의 변경 이유: 각자의 비즈니스 규칙
Use Case의 변경 이유: 도메인 간 협력 방식

→ 각 도메인은 독립적
→ Use Case는 이들을 조율
```

**예시 1: 주문 생성**

```
Domain Layer:
order/domain/Order.java
product/domain/Product.java
user/domain/User.java
inventory/domain/Inventory.java

Application Layer:
order/application/CreateOrderUseCase.java

@UseCase
class CreateOrderUseCase {
    // 여러 도메인 협력
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    private final InventoryRepository inventoryRepository;

    public OrderResult execute(CreateOrderCommand command) {
        // 1. User 도메인
        User user = userRepository.findById(command.getUserId());
        user.validateCanOrder();  // User의 비즈니스 규칙

        // 2. Product 도메인
        List<Product> products = productRepository.findAllById(
            command.getProductIds()
        );
        products.forEach(Product::validateAvailable);  // Product의 규칙

        // 3. Inventory 도메인
        inventoryRepository.reserveStock(products);  // Inventory의 규칙

        // 4. Order 도메인
        Order order = Order.create(user, products);  // Order의 규칙

        return orderRepository.save(order);
    }
}

변경 시나리오:
"User 검증 규칙 변경" → User 도메인만 변경
"Product 재고 정책 변경" → Product 도메인만 변경
"Order 생성 규칙 변경" → Order 도메인만 변경
"주문 생성 프로세스 변경" → CreateOrderUseCase만 변경
```

**왜 이렇게 설계하는가?**

```
각 도메인은 독립적인 변경 이유:
- User: "사용자 검증 규칙"
- Product: "상품 판매 가능 조건"
- Inventory: "재고 관리 정책"
- Order: "주문 생성 규칙"

Use Case의 변경 이유:
- "도메인 간 협력 순서"
- "어떤 도메인을 사용할지"

→ N개의 독립적인 도메인
→ 1개의 조율 Use Case
```

**예시 2: 정산**

```
Domain Layer:
order/domain/Order.java
payment/domain/Payment.java
seller/domain/Seller.java
commission/domain/CommissionPolicy.java

Application Layer:
settlement/application/CalculateSettlementUseCase.java

@UseCase
class CalculateSettlementUseCase {
    public SettlementResult execute(CalculateCommand command) {
        // 1. Order 도메인
        List<Order> orders = orderRepository.findCompletedOrders(
            command.getPeriod()
        );

        // 2. Payment 도메인
        List<Payment> payments = paymentRepository.findByOrders(orders);
        Money totalSales = Payment.calculateTotal(payments);

        // 3. CommissionPolicy 도메인
        Money commission = commissionPolicy.calculate(totalSales);

        // 4. Seller 도메인
        Money settlement = totalSales.subtract(commission);
        seller.addSettlement(settlement);

        return SettlementResult.of(totalSales, commission, settlement);
    }
}

변경 시나리오:
"수수료 정책 변경" → CommissionPolicy만 변경
"정산 계산 방식 변경" → Use Case만 변경
```

### 매핑 판단 결정 트리

```
┌──────────────────────────────┐
│ Use Case를 설계한다          │
└────────────┬─────────────────┘
             ▼
┌──────────────────────────────┐
│ 비즈니스 규칙이 있는가?      │
└───┬──────────────────┬───────┘
    │ Yes              │ No
    ▼                  ▼
┌────────────┐   ┌──────────────┐
│ Domain 필요│   │ 0 Domain :   │
└─────┬──────┘   │ 1 Application│
      ▼          └──────────────┘
┌──────────────────────────────┐
│ 규칙이 여러 Use Case에서     │
│ 재사용되는가?                │
└───┬──────────────────┬───────┘
    │ Yes              │ No
    ▼                  ▼
┌────────────┐   ┌──────────────┐
│ 1 Domain : │   │ Domain은     │
│ N App      │   │ Use Case에만 │
└────────────┘   └──────────────┘
      ▼
┌──────────────────────────────┐
│ 여러 도메인이 협력하는가?    │
└───┬──────────────────┬───────┘
    │ Yes              │ No
    ▼                  ▼
┌────────────┐   ┌──────────────┐
│ N Domain : │   │ 1 Domain :   │
│ 1 App      │   │ 1 App        │
└────────────┘   └──────────────┘
```

---

## 실전 설계 판단 사례

### 사례 1: 회원 등급 시스템

**요구사항**:

```
- 회원 등급: BRONZE, SILVER, GOLD, VIP
- 등급별 혜택이 다름
- 구매 금액에 따라 등급 자동 승급
```

**잘못된 설계 (기능 기준)**:

```
user/
├── User.java
├── UserGrade.java
├── UserGradeService.java      # 등급 관련 모든 것
└── UserGradeRepository.java

문제:
"등급 혜택 변경"과 "등급 승급 조건 변경"이 같은 클래스 수정
→ 다른 이유의 변경이 섞임
```

**올바른 설계 (변경 이유 기준)**:

```
# Domain Layer
user/domain/
├── User.java
├── UserGrade.java
└── GradePolicy.java           # 등급 혜택 규칙
    - getDiscountRate(grade)
    - getPointRate(grade)

user/domain/
└── GradePromotionPolicy.java  # 등급 승급 규칙
    - canPromote(user)
    - calculateNextGrade(purchaseAmount)

# Application Layer
user/application/
├── PromoteUserGradeUseCase.java    # 등급 승급
└── ApplyGradeBenefitUseCase.java   # 혜택 적용

변경 시나리오:
"VIP 할인율 15% → 20%" → GradePolicy만 변경
"승급 조건 변경" → GradePromotionPolicy만 변경
"승급 프로세스 변경" → PromoteUserGradeUseCase만 변경
```

**판단 과정**:

```
1. 변경 시나리오 식별
   - 등급 혜택 변경 (할인율, 포인트율)
   - 등급 승급 조건 변경 (필요 구매 금액)
   - 등급 승급 프로세스 변경 (알림, 쿠폰 지급)

2. 변경 이유 분류
   - 등급 혜택: 비즈니스 규칙 → Domain
   - 승급 조건: 비즈니스 규칙 → Domain
   - 승급 프로세스: 실행 방식 → Application

3. 응집도 검증
   - 등급 혜택과 승급 조건: 다른 이유로 변경 → 분리
   - 각각 독립적인 Policy로
```

### 사례 2: 포인트 시스템

**요구사항**:

```
- 구매 시 포인트 적립
- 포인트로 결제 가능
- 포인트 소멸
- 포인트 선물
```

**분석**:

```
질문 1: 이것들이 같은 이유로 변경되는가?

"포인트 적립률 변경" → 적립 로직만
"포인트 결제 규칙 변경" → 결제 로직만
"포인트 소멸 정책 변경" → 소멸 로직만

→ No, 다른 이유로 변경됨

질문 2: Domain이 필요한가?

"포인트 적립률 계산" → 비즈니스 규칙 → Domain
"포인트 사용 규칙" → 비즈니스 규칙 → Domain
"포인트 소멸 조건" → 비즈니스 규칙 → Domain

→ Yes, Domain 필요
```

**설계**:

```
# Domain Layer
point/domain/
├── Point.java                 # 포인트 엔티티
├── PointTransaction.java      # 포인트 거래
├── EarnPolicy.java            # 적립 정책
├── UsePolicy.java             # 사용 정책
└── ExpiryPolicy.java          # 소멸 정책

# Application Layer
point/application/
├── EarnPointUseCase.java      # 포인트 적립
├── UsePointUseCase.java       # 포인트 사용
├── ExpirePointUseCase.java    # 포인트 소멸
└── GiftPointUseCase.java      # 포인트 선물

매핑: 1 Domain (Point) : N Application

변경 시나리오:
"적립률 변경" → EarnPolicy만
"사용 제한 변경" → UsePolicy만
"소멸 기간 변경" → ExpiryPolicy만
"포인트 적립 프로세스 변경" → EarnPointUseCase만
```

### 사례 3: 주문 조회 API

**요구사항**:

```
- 주문 목록 조회
- 주문 상세 조회
- 엑셀 다운로드
```

**분석**:

```
질문: 비즈니스 규칙이 있는가?

"주문 목록 조회" → 단순 조회 (규칙 없음)
"주문 상세 조회" → 단순 조회 (규칙 없음)
"엑셀 다운로드" → 데이터 변환 (규칙 없음)

→ No, 비즈니스 규칙 없음
```

**설계**:

```
# Domain Layer
(없음 - 기존 Order 도메인 재사용)

# Application Layer
order/application/query/
├── GetOrderListUseCase.java
├── GetOrderDetailUseCase.java
└── ExportOrderExcelUseCase.java

매핑: 0 Domain : N Application

왜?
- 조회/변환만 하는 Use Case
- 새로운 비즈니스 규칙 없음
- 기존 Order 도메인 활용

변경 시나리오:
"조회 조건 추가" → GetOrderListUseCase만
"엑셀 형식 변경" → ExportOrderExcelUseCase만
```

### 사례 4: 주문-결제-배송 통합

**요구사항**:

```
- 주문 생성 시 결제 처리
- 결제 완료 시 배송 시작
- 배송 완료 시 주문 완료
```

**분석**:

```
질문: 각 도메인의 변경 이유는?

Order: "주문 생성/취소 규칙"
Payment: "결제 승인/취소 규칙"
Delivery: "배송 처리 규칙"

→ 서로 다른 변경 이유

질문: 하나의 Use Case로?

"주문 생성" Use Case가:
- Order 규칙 사용
- Payment 규칙 사용
- Delivery 규칙 사용 (예약)

→ 여러 도메인 협력
```

**설계**:

```
# Domain Layer
order/domain/Order.java
payment/domain/Payment.java
delivery/domain/Delivery.java

# Application Layer
order/application/
└── CreateOrderUseCase.java

@UseCase
class CreateOrderUseCase {
    private final OrderRepository orderRepository;
    private final PaymentProcessor paymentProcessor;
    private final DeliveryScheduler deliveryScheduler;

    public OrderResult execute(CreateOrderCommand command) {
        // 1. 주문 생성 (Order 도메인)
        Order order = Order.create(command);
        orderRepository.save(order);

        // 2. 결제 처리 (Payment 도메인)
        Payment payment = paymentProcessor.process(order);

        // 3. 배송 예약 (Delivery 도메인)
        deliveryScheduler.schedule(order, payment);

        return OrderResult.from(order);
    }
}

매핑: N Domain : 1 Application

변경 시나리오:
"주문 생성 규칙 변경" → Order만
"결제 처리 방식 변경" → Payment만
"배송 스케줄링 변경" → Delivery만
"전체 프로세스 순서 변경" → CreateOrderUseCase만
```

---

## 리팩토링 트리거

### 언제 구조를 바꿔야 하는가?

**트리거 1: 변경이 여러 곳에 분산**

```
증상:
"할인율 변경"을 위해 5개 클래스를 수정함

원인:
할인 로직이 중복되어 분산됨

리팩토링:
DiscountPolicy로 추출
→ 1곳만 수정하도록

Before:
CreateOrderUseCase {
    if (user.isVIP()) discount = price * 0.15;
}
CancelOrderUseCase {
    if (user.isVIP()) refund = price * 0.85;
}

After:
DiscountPolicy {
    Money calculate(User user, Money price) {
        if (user.isVIP()) return price.multiply(0.15);
    }
}
```

**트리거 2: 서로 다른 이유로 함께 변경**

```
증상:
"이메일 템플릿 변경"과 "주문 취소 규칙 변경"이
같은 클래스를 수정함

원인:
다른 변경 이유가 같은 곳에 섞임

리팩토링:
Use Case 분리

Before:
CancelOrderService {
    void cancel() {
        order.cancel();        // 도메인 규칙
        sendEmail();           // 알림 로직
        updateInventory();     // 재고 로직
    }
}

After:
CancelOrderUseCase {
    void execute() {
        order.cancel();
        eventPublisher.publish(new OrderCancelledEvent());
    }
}

SendOrderCancelNotificationHandler {
    void handle(OrderCancelledEvent event) {
        sendEmail(event);
    }
}
```

**트리거 3: 테스트가 어려움**

```
증상:
Order를 테스트하려면 DB, 외부 API를 모두 준비해야 함

원인:
Domain이 Infrastructure에 의존

리팩토링:
의존성 역전 (Port 도입)

Before:
Order {
    void create() {
        paymentApi.call();     // 직접 의존
        emailSender.send();    // 직접 의존
    }
}

After:
Order {
    void create() {
        // 순수 비즈니스 로직만
    }
}

CreateOrderUseCase {
    void execute() {
        order.create();
        paymentProcessor.process(order);  // Port
        eventPublisher.publish(event);    // Port
    }
}
```

**트리거 4: 변경 영향도 예측 불가**

```
증상:
"이 코드를 변경하면 어디에 영향을 주는지 모르겠음"

원인:
순환 의존성, 양방향 의존성

리팩토링:
단방향 의존성으로 정리

Before:
Order ←→ Payment  (순환 의존)

After:
Order → PaymentProcessor (Port)
         ↑
    PaymentAdapter (Infrastructure)
```

### 리팩토링 우선순위

```
우선순위 1 (즉시):
- 순환 의존성
- Domain이 Infrastructure 의존
- 하나의 변경이 10개 이상 파일 수정

우선순위 2 (중요):
- 같은 로직이 3곳 이상 중복
- 하나의 클래스가 3개 이상의 이유로 변경
- 테스트 작성이 어려움

우선순위 3 (개선):
- 클래스/메서드가 지나치게 큼
- 변경 영향도 파악이 어려움
- 새로운 기능 추가가 어려움
```

---

## 체크리스트

### 설계 검증

- [ ] 각 모듈의 변경 이유를 1문장으로 설명할 수 있는가?
- [ ] 하나의 변경 시나리오가 하나의 모듈만 영향을 주는가?
- [ ] Domain 클래스가 Infrastructure를 의존하지 않는가?
- [ ] Application Use Case가 명확한 비즈니스 목표를 표현하는가?

### Domain 레이어

- [ ] 이 코드는 UI/프레임워크가 바뀌어도 동일한가?
- [ ] 이 규칙은 어떤 Use Case에서든 동일한가?
- [ ] 외부 의존성 없이 테스트 가능한가?

### Application 레이어

- [ ] 이 Use Case는 하나의 비즈니스 목표를 달성하는가?
- [ ] 도메인 규칙을 직접 구현하지 않고 조율만 하는가?
- [ ] 트랜잭션 경계가 명확한가?

### 매핑 패턴

- [ ] Domain 없이 Application만 있다면, 정말 비즈니스 규칙이 없는가?
- [ ] 1:N 매핑이라면, 각 Use Case의 변경 이유가 다른가?
- [ ] N:1 매핑이라면, 각 Domain의 변경 이유가 독립적인가?

---

이 가이드는 "무엇을 어디에 넣을지"가 아니라 "왜 이렇게 나누는지"를 이해하고 판단하는 데 초점을 맞춥니다. 변경 이유를 기준으로 설계하면 자연스럽게 응집도 높고 결합도 낮은 구조가 만들어집니다.
