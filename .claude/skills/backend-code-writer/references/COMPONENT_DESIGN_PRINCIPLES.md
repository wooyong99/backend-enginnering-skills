# 컴포넌트 설계 원칙 가이드

> **목적**: 효과적인 컴포넌트 구성과 의존성 관리  
> **대상**: 백엔드 개발자 및 AI 개발 어시스턴트  
> **기반**: SOLID 원칙 + 컴포넌트 원칙 (Robert C. Martin)

---

## 📋 목차

1. [컴포넌트 정의](#컴포넌트-정의)
2. [컴포넌트 응집도 원칙](#컴포넌트-응집도-원칙)
3. [컴포넌트 결합 원칙](#컴포넌트-결합-원칙)
4. [실전 적용 가이드](#실전-적용-가이드)
5. [의존성 관리 전략](#의존성-관리-전략)
6. [컴포넌트 분리 패턴](#컴포넌트-분리-패턴)
7. [안티패턴과 해결책](#안티패턴과-해결책)

---

## 컴포넌트 정의

### 컴포넌트란?

**컴포넌트**: 배포 가능한 가장 작은 단위

**백엔드 시스템에서의 컴포넌트**:

```
레벨 1: 모듈/패키지
- domain.order
- domain.payment
- infrastructure.client

레벨 2: 라이브러리/JAR
- order-core.jar
- payment-core.jar
- common-domain.jar

레벨 3: 마이크로서비스
- order-service
- payment-service
- user-service
```

**컴포넌트 경계 판단 기준**:

```
1. 배포 단위
   - 독립적으로 배포 가능한가?
   - 버전 관리가 필요한가?

2. 변경 빈도
   - 함께 변경되는가?
   - 독립적으로 변경되는가?

3. 재사용성
   - 다른 곳에서 재사용되는가?
   - 독립적으로 사용 가능한가?

4. 책임
   - 단일 책임을 가지는가?
   - 응집도가 높은가?
```

---

## 컴포넌트 응집도 원칙

> **목적**: "어떤 클래스들을 같은 컴포넌트로 묶을 것인가?"

### 1. REP (Reuse/Release Equivalence Principle)

### 재사용/릴리스 등가 원칙

**원칙**: 재사용 단위는 릴리스 단위와 같다.

**핵심 개념**:

```
- 컴포넌트는 응집성 있는 클래스와 모듈들로 구성
- 컴포넌트를 구성하는 모든 모듈은 서로 공유하는 주제나 목적이 있어야 함
- 함께 릴리스될 수 없다면 같은 컴포넌트에 속하면 안 됨
```

**올바른 예시**:

```
✅ GOOD: 응집된 컴포넌트

common-money/
├── Money.java
├── Currency.java
├── MoneyCalculator.java
└── ExchangeRate.java

→ 모두 "금액 처리"라는 단일 주제
→ 함께 버전 관리됨
→ 함께 릴리스됨
```

**잘못된 예시**:

```
❌ BAD: 비응집 컴포넌트

common-utils/
├── Money.java          // 금액
├── StringUtils.java    // 문자열
├── DateUtils.java      // 날짜
└── EmailSender.java    // 이메일

→ 서로 관련 없는 기능들
→ Money만 변경되어도 전체 릴리스 필요
→ StringUtils만 필요해도 전체를 의존해야 함
```

**적용 방법**:

```
1. 주제별로 그룹화
   common-money/       # 금액 관련
   common-datetime/    # 날짜/시간 관련
   common-validation/  # 검증 관련

2. 버전 관리
   common-money:1.0.0
   common-datetime:2.1.0

3. 변경 추적
   - Money 변경 → common-money만 새 버전
   - 다른 컴포넌트는 영향 없음
```

**실전 예시**:

```
domain-common/
├── value/
│   ├── Money.java
│   ├── Email.java
│   ├── PhoneNumber.java
│   └── Address.java
└── event/
    └── DomainEvent.java

→ "공통 도메인 개념"이라는 명확한 주제
→ 함께 버전 관리
→ 의미 있는 릴리스 단위
```

### 2. CCP (Common Closure Principle)

### 공통 폐쇄 원칙

**원칙**: 동일한 이유로 동일한 시점에 변경되는 클래스를 같은 컴포넌트로 묶어라.

**핵심 개념**:

```
- 함께 변경되는 것들은 함께 묶어라
- SRP의 컴포넌트 버전
- 변경의 영향을 최소화
```

**올바른 예시**:

```
✅ GOOD: 함께 변경되는 클래스들

order-domain/
├── Order.java
├── OrderItem.java
├── OrderStatus.java
└── OrderPolicy.java

→ "주문" 비즈니스 변경 시 함께 변경됨
→ 변경 영향이 이 컴포넌트에만 국한됨
```

**잘못된 예시**:

```
❌ BAD: 다른 이유로 변경되는 클래스들

business-logic/
├── OrderCalculator.java    # 주문 정책 변경
├── PaymentValidator.java   # 결제 규칙 변경
├── UserAuthenticator.java  # 인증 정책 변경
└── ProductPricer.java      # 가격 정책 변경

→ 각각 다른 이유로 변경됨
→ OrderCalculator 변경 시 전체 컴포넌트 재배포
→ 불필요한 의존성 발생
```

**변경 시나리오 분석**:

```
시나리오 1: "주문 취소 정책 변경"
영향받는 클래스:
- Order.java (취소 로직)
- OrderStatus.java (상태 추가)
- OrderPolicy.java (취소 규칙)

✅ 모두 order-domain에 있다면:
→ order-domain만 변경
→ 다른 컴포넌트 영향 없음

❌ 여러 컴포넌트에 분산되어 있다면:
→ 여러 컴포넌트 변경 필요
→ 의존성 버전 관리 복잡
```

**적용 방법**:

```
1. 변경 이유로 그룹화

order-pricing/
├── PriceCalculator.java
├── DiscountPolicy.java
└── PricingRule.java
→ "가격 정책 변경" 시 함께 변경

order-inventory/
├── StockChecker.java
├── ReservationManager.java
└── StockPolicy.java
→ "재고 정책 변경" 시 함께 변경

2. 변경 영향 최소화
- 가격 정책 변경 → order-pricing만 변경
- 재고 정책 변경 → order-inventory만 변경
```

**실전 예시**:

```
payment-pg/
├── PaymentGateway.java
├── PaymentRequest.java
├── PaymentResponse.java
└── PaymentMapper.java

→ "PG사 연동 규격 변경" 시 함께 변경
→ 변경 영향이 이 컴포넌트에만 국한
```

### 3. CRP (Common Reuse Principle)

### 공통 재사용 원칙

**원칙**: 컴포넌트 사용자들을 필요하지 않은 것에 의존하게 강요하지 마라.

**핵심 개념**:

```
- 함께 재사용되는 클래스들을 같은 컴포넌트로 묶어라
- 컴포넌트 내의 클래스들은 서로 분리될 수 없어야 함
- ISP(Interface Segregation Principle)의 컴포넌트 버전
```

**올바른 예시**:

```
✅ GOOD: 함께 사용되는 클래스들

order-core/
├── Order.java
├── OrderItem.java
└── OrderStatus.java

→ Order를 사용하려면 OrderItem, OrderStatus도 필요
→ 항상 함께 사용됨
→ 분리할 수 없음
```

**잘못된 예시**:

```
❌ BAD: 따로 사용되는 클래스들

common-models/
├── Order.java          # 주문 도메인
├── User.java           # 사용자 도메인
├── Product.java        # 상품 도메인
├── Payment.java        # 결제 도메인
└── Shipping.java       # 배송 도메인

→ Order만 필요해도 전체를 의존해야 함
→ Product 변경 시 Order를 사용하는 모든 곳에 영향
→ 불필요한 재배포 발생
```

**의존성 영향 분석**:

```
시나리오: "Product 클래스 변경"

❌ BAD (하나의 컴포넌트):
common-models (v1.0.0) 변경
  ↓
common-models (v1.1.0) 릴리스
  ↓
Order만 사용하는 서비스도 업데이트 필요
  ↓
불필요한 재배포 발생

✅ GOOD (분리된 컴포넌트):
product-domain (v1.0.0) 변경
  ↓
product-domain (v1.1.0) 릴리스
  ↓
product-domain을 사용하는 서비스만 업데이트
  ↓
order-domain은 영향 없음
```

**적용 방법**:

```
1. 사용 패턴 분석

# 함께 사용됨
order-domain/
├── Order.java
├── OrderItem.java
└── OrderStatus.java

# 독립적으로 사용됨
user-domain/
├── User.java
├── UserProfile.java
└── UserStatus.java

2. 의존성 최소화

order-service가 필요한 것:
✅ order-domain (Order, OrderItem, OrderStatus)
✅ user-domain (UserId만) → 인터페이스로 분리
❌ user-domain (User 전체) → 불필요
```

**실전 예시**:

```
# 잘못된 구조
common/
├── Money.java          # 모든 곳에서 사용
├── Email.java          # 사용자 도메인만 사용
├── Address.java        # 배송 도메인만 사용
└── PaymentMethod.java  # 결제 도메인만 사용

→ Money만 필요해도 전체 의존
→ Email 변경 시 전체 영향

# 올바른 구조
common-value/
└── Money.java          # 공통으로 사용

user-domain/
├── User.java
└── Email.java          # 함께 사용됨

shipping-domain/
├── Delivery.java
└── Address.java        # 함께 사용됨
```

### 응집도 원칙 간의 긴장 관계

```
REP ←→ CCP ←→ CRP

REP: 재사용을 위해 작게 나눠라
CCP: 유지보수를 위해 크게 묶어라
CRP: 사용하지 않는 것에 의존하지 마라

→ 균형점을 찾아야 함
→ 프로젝트 단계에 따라 다름
```

**프로젝트 단계별 전략**:

```
초기 개발 단계:
CCP 우선 → 변경 용이성
- 빠른 기능 추가
- 자주 변경됨
- 크게 묶어서 개발

성숙 단계:
CRP/REP 우선 → 재사용성/안정성
- 기능 안정화
- 재사용 증가
- 작게 나눠서 관리
```

---

## 컴포넌트 결합 원칙

> **목적**: "컴포넌트 간 관계를 어떻게 설정할 것인가?"

### 1. ADP (Acyclic Dependencies Principle)

### 의존성 비순환 원칙

**원칙**: 컴포넌트 의존성 그래프에 순환이 있어서는 안 된다.

**핵심 개념**:

```
- 순환 의존성은 "아침이 오지 않는 증후군" 발생
- 한 컴포넌트 변경이 다른 모든 컴포넌트에 영향
- 빌드, 테스트, 배포가 복잡해짐
```

**순환 의존성의 문제**:

```
❌ BAD: 순환 의존성

order-domain
    ↓ (의존)
payment-domain
    ↓ (의존)
voucher-domain
    ↓ (의존)
order-domain ← 순환!

문제점:
1. order-domain 변경
   → payment-domain 재빌드 필요
   → voucher-domain 재빌드 필요
   → order-domain 재빌드 필요 (순환!)

2. 독립적인 테스트 불가능

3. 릴리스 순서 결정 불가능

4. 이해하기 어려움
```

**순환 끊기 방법 1: 의존성 역전 (DIP)**

```
❌ BEFORE: 순환 의존

order-domain
  ↓
payment-domain
  ↓
order-domain (Order 클래스 필요)

✅ AFTER: DIP 적용

order-domain
  ↓
payment-domain
  ↑ (implements)
order-domain/port/PaymentProcessor (인터페이스)

의존성 방향:
order-domain → payment-domain (인터페이스)
payment-domain → order-domain/port (인터페이스 구현)
```

**실전 예시**:

```
# BEFORE: 순환 의존
OrderService {
    private PaymentService paymentService;  // order → payment
}

PaymentService {
    private OrderRepository orderRepository;  // payment → order
}

# AFTER: 이벤트로 분리
OrderService {
    private EventPublisher eventPublisher;

    void createOrder() {
        // ...
        eventPublisher.publish(new OrderCreatedEvent());
    }
}

PaymentEventHandler {
    @EventListener
    void handle(OrderCreatedEvent event) {
        // 결제 처리
    }
}

의존성:
order → event
payment → event
(순환 없음)
```

**순환 끊기 방법 2: 새로운 컴포넌트 생성**

```
❌ BEFORE: 순환 의존

user-domain
  ↓
order-domain
  ↓
user-domain (User 정보 필요)

✅ AFTER: 공통 컴포넌트 추출

common-domain (UserId, OrderId)
  ↑           ↑
user-domain  order-domain

공통으로 사용되는 개념을 별도 컴포넌트로:
- UserId (값 객체)
- OrderId (값 객체)
- 기본 인터페이스
```

**순환 끊기 방법 3: 중재자 패턴**

```
❌ BEFORE: 양방향 의존

order-domain ↔ inventory-domain

✅ AFTER: 중재자 도입

order-domain
  ↓
order-inventory-coordinator (중재자)
  ↓
inventory-domain

중재자가 조율:
- 주문 생성 시 재고 확인
- 재고 변경 알림
```

### 2. SDP (Stable Dependencies Principle)

### 안정된 의존성 원칙

**원칙**: 안정성의 방향으로 의존하라.

**안정성 측정**:

```
I = Fan-out / (Fan-in + Fan-out)

Fan-in: 이 컴포넌트에 의존하는 클래스 수
Fan-out: 이 컴포넌트가 의존하는 클래스 수

I = 0: 최대 안정 (많이 의존됨, 의존 없음)
I = 1: 최대 불안정 (의존됨 없음, 많이 의존)
```

**안정성 계산 예시**:

```
domain-common
  ↑ (의존됨)
  ├─ order-domain
  ├─ payment-domain
  ├─ user-domain
  └─ product-domain

Fan-in = 4 (4개 컴포넌트가 의존)
Fan-out = 0 (다른 컴포넌트 의존 없음)
I = 0 / (4 + 0) = 0 (최대 안정)

→ 변경이 어려움 (많은 곳에 영향)
→ 변경되어서는 안 됨
→ 추상적이고 안정적인 개념만 포함
```

**올바른 의존성 방향**:

```
✅ GOOD: 안정 → 불안정 (X)
        불안정 → 안정 (O)

불안정한 컴포넌트 (I = 0.8)
  ↓ (의존)
안정적인 컴포넌트 (I = 0.2)

예시:
order-usecase (I = 0.9, 자주 변경)
  ↓
order-domain (I = 0.1, 안정적)
  ↓
common-value (I = 0.0, 매우 안정적)
```

**잘못된 의존성 방향**:

```
❌ BAD: 안정 → 불안정

common-domain (I = 0.1, 안정적)
  ↓
payment-external-api (I = 0.9, 자주 변경)

문제:
- payment-external-api가 변경되면
- common-domain도 영향받음
- common-domain에 의존하는 모든 곳 영향
```

**해결 방법**:

```
✅ GOOD: DIP 적용

common-domain (I = 0.1)
  ↑
payment-port (interface)
  ↑
payment-external-api (I = 0.9)

안정적인 컴포넌트가 인터페이스 소유
불안정한 컴포넌트가 인터페이스 구현
```

**실전 적용**:

```
# 안정성 순서 (안정 → 불안정)

1. common-value (I ≈ 0.0)
   - Money, Email, Phone 등
   - 절대 변경되면 안 됨

2. domain-core (I ≈ 0.1)
   - Order, User, Product 등
   - 안정적인 비즈니스 개념

3. domain-port (I ≈ 0.2)
   - Repository, Client 인터페이스
   - 상대적으로 안정적

4. usecase (I ≈ 0.7)
   - 비즈니스 로직 조율
   - 자주 변경됨

5. infrastructure (I ≈ 0.9)
   - 외부 시스템 연동
   - 매우 자주 변경됨

의존성 방향:
infrastructure → usecase → domain-core → common-value
```

### 3. SAP (Stable Abstractions Principle)

### 안정된 추상화 원칙

**원칙**: 컴포넌트는 안정된 정도만큼 추상화되어야 한다.

**추상화 정도 측정**:

```
A = 추상 클래스/인터페이스 수 / 전체 클래스 수

A = 0: 구체적 (추상 클래스 없음)
A = 1: 추상적 (모두 추상 클래스/인터페이스)
```

**주요 구역**:

```
I (불안정성)
↑
│  고통의 구역
│  (I=0, A=0)
│  ┌────────────
│  │ 안정적이지만
│  │ 구체적
│  │ 변경 어려움
│
├──────────────────→ A (추상화)
│        쓸모없는 구역
│        (I=1, A=1)
│        ──────────┐
│        추상적이지만│
│        사용되지 않음│
│                  │
│
│  주계열 (이상적)
│  I + A = 1
│
▼
```

**올바른 조합**:

```
✅ GOOD: 안정적 + 추상적

domain-port (I = 0.1, A = 0.9)
├── OrderRepository (interface)
├── PaymentProcessor (interface)
└── EventPublisher (interface)

→ 안정적 (많이 의존됨)
→ 추상적 (인터페이스/추상 클래스)
→ 확장 가능
```

```
✅ GOOD: 불안정 + 구체적

infrastructure (I = 0.9, A = 0.1)
├── JpaOrderRepository (구체 클래스)
├── RestPaymentClient (구체 클래스)
└── KafkaEventPublisher (구체 클래스)

→ 불안정 (자주 변경)
→ 구체적 (구현체)
→ 변경 용이
```

**잘못된 조합**:

```
❌ BAD: 안정적 + 구체적 (고통의 구역)

common-utils (I = 0.0, A = 0.0)
├── StringUtils (구체 클래스)
├── DateUtils (구체 클래스)
└── FileUtils (구체 클래스)

→ 매우 안정적 (모든 곳에서 의존)
→ 매우 구체적 (변경 어려움)
→ 확장 불가능
→ "고통의 구역"
```

```
❌ BAD: 불안정 + 추상적 (쓸모없는 구역)

unused-interfaces (I = 1.0, A = 1.0)
├── SomeInterface (사용 안 됨)
└── AnotherInterface (사용 안 됨)

→ 매우 불안정 (아무도 의존 안 함)
→ 매우 추상적 (인터페이스만)
→ "쓸모없는 구역"
```

**실전 적용**:

```
# 레이어별 I, A 값

Layer 1: Domain Core (안정 + 추상)
- I = 0.1 (매우 안정)
- A = 0.7 (상당히 추상적)
- 추상 클래스, 인터페이스 많음

Layer 2: Application Port (안정 + 추상)
- I = 0.2 (안정)
- A = 0.9 (매우 추상적)
- 거의 인터페이스만

Layer 3: Use Case (중간)
- I = 0.5 (중간)
- A = 0.3 (약간 구체적)
- 구체적인 로직 + 일부 추상화

Layer 4: Infrastructure (불안정 + 구체)
- I = 0.9 (매우 불안정)
- A = 0.1 (매우 구체적)
- 구현체만
```

**SAP 위반 해결**:

```
❌ 문제: 안정적이지만 구체적

common-utils (I = 0.0, A = 0.0)

✅ 해결 1: 추상화 도입

common-utils-api (I = 0.0, A = 1.0)
└── StringFormatter (interface)

common-utils-impl (I = 0.9, A = 0.0)
└── DefaultStringFormatter (구체 클래스)

✅ 해결 2: 덜 안정적으로

- 의존성 줄이기
- 필요한 곳에만 노출
```

---

## 실전 적용 가이드

### 컴포넌트 분리 의사결정 프로세스

```
1단계: 응집도 평가
┌─────────────────────────────────┐
│ 함께 변경되는가? (CCP)          │
│ 함께 재사용되는가? (CRP)        │
│ 함께 릴리스되는가? (REP)        │
└────────┬────────────────────────┘
         │
    ┌────┴────┐
    Yes      No
     │        │
     ▼        ▼
  같은     다른
  컴포넌트  컴포넌트

2단계: 의존성 검증
┌─────────────────────────────────┐
│ 순환 의존성 있는가? (ADP)       │
│ 안정성 방향 맞는가? (SDP)       │
│ 추상화 수준 맞는가? (SAP)       │
└────────┬────────────────────────┘
         │
    ┌────┴────┐
    문제      OK
    있음      │
     │        ▼
     ▼     완료
  리팩토링
```

### 패키지 구조 예시

**모놀리식 구조** (단일 프로젝트):

```
project/
├── common/
│   ├── value/                    # I=0.0, A=0.8
│   │   ├── Money
│   │   ├── Email
│   │   └── Phone
│   └── event/                    # I=0.0, A=1.0
│       └── DomainEvent
│
├── domain/
│   ├── order/                    # I=0.2, A=0.6
│   │   ├── Order
│   │   ├── OrderItem
│   │   └── OrderStatus
│   ├── payment/                  # I=0.2, A=0.6
│   └── user/                     # I=0.1, A=0.5
│
├── application/
│   ├── order/                    # I=0.5, A=0.3
│   │   ├── usecase/
│   │   ├── port/                 # I=0.3, A=1.0
│   │   └── validator/
│   └── payment/                  # I=0.5, A=0.3
│
└── infrastructure/
    ├── persistence/              # I=0.9, A=0.1
    ├── client/                   # I=0.9, A=0.1
    └── event/                    # I=0.9, A=0.1

의존성 방향:
infrastructure → application → domain → common
     (불안정)                             (안정)
     (구체적)                             (추상적)
```

**멀티 모듈 구조**:

```
project/
├── common-domain/                # I=0.0, A=0.8
│   └── src/main/java/
│       ├── Money
│       ├── Email
│       └── DomainEvent
│
├── order-domain/                 # I=0.2, A=0.6
│   └── src/main/java/
│       ├── Order
│       └── OrderRepository (interface)
│
├── order-application/            # I=0.5, A=0.3
│   └── src/main/java/
│       └── CreateOrderUseCase
│
└── order-infrastructure/         # I=0.9, A=0.1
    └── src/main/java/
        └── JpaOrderRepository

의존성:
order-infrastructure → order-application → order-domain → common-domain
```

### 변경 시나리오별 영향도 분석

**시나리오 1: 도메인 규칙 변경**

```
변경: "주문 취소 정책 변경"

영향받는 컴포넌트:
✅ order-domain (Order.cancel() 메서드)
   └─ I = 0.2 (비교적 안정적)

영향 없는 컴포넌트:
○ payment-domain
○ user-domain
○ infrastructure

→ 단일 컴포넌트만 변경 (CCP 준수)
```

**시나리오 2: 외부 API 변경**

```
변경: "결제 PG사 API 스펙 변경"

영향받는 컴포넌트:
✅ payment-infrastructure (PaymentClient)
   └─ I = 0.9 (매우 불안정, 변경 쉬움)

영향 없는 컴포넌트:
○ payment-domain (인터페이스로 격리됨)
○ payment-application

→ 가장 불안정한 컴포넌트만 변경 (SDP 준수)
```

**시나리오 3: 공통 값 객체 변경**

```
변경: "Money 클래스에 메서드 추가"

주의:
❌ common-value는 I = 0.0 (매우 안정적)
   → 많은 컴포넌트가 의존
   → 신중하게 변경해야 함

전략:
✅ 하위 호환성 유지
✅ 새 메서드 추가만
✅ 기존 메서드 변경 금지
✅ 충분한 테스트
```

---

## 의존성 관리 전략

### 의존성 규칙

**레이어별 의존성 허용 범위**:

```
Infrastructure Layer:
✅ Application Layer 의존 가능
✅ Domain Layer 의존 가능
✅ 외부 라이브러리 의존 가능

Application Layer:
✅ Domain Layer 의존 가능
❌ Infrastructure Layer 의존 금지

Domain Layer:
❌ Application Layer 의존 금지
❌ Infrastructure Layer 의존 금지
✅ Common 의존 가능
```

**컴포넌트 간 의존성 관리**:

```
1. 직접 의존 (Direct Dependency)
order-usecase → order-domain

2. 인터페이스 의존 (Interface Dependency)
order-usecase → order-port (interface)
order-infrastructure → order-port (implementation)

3. 이벤트 의존 (Event Dependency)
order-usecase → event
payment-handler ← event

4. 공통 의존 (Common Dependency)
order-domain → common-value
payment-domain → common-value
```

### 의존성 버전 관리

**시맨틱 버저닝**:

```
MAJOR.MINOR.PATCH

MAJOR: 하위 호환 안 되는 변경
MINOR: 하위 호환되는 기능 추가
PATCH: 하위 호환되는 버그 수정

예시:
common-value: 1.2.3
  1 = Major (인터페이스 변경)
  2 = Minor (새 메서드 추가)
  3 = Patch (버그 수정)
```

**안정적 컴포넌트 버전 관리**:

```
# 안정적 컴포넌트 (I < 0.3)
common-value: 1.0.0 → 1.0.1 (PATCH만)
domain-core: 1.2.0 → 1.3.0 (MINOR만)

→ MAJOR 버전업 최대한 지양
→ 하위 호환성 필수

# 불안정 컴포넌트 (I > 0.7)
infrastructure: 1.0.0 → 2.0.0 (MAJOR 가능)

→ 자유로운 변경 가능
→ 다른 컴포넌트 영향 최소
```

---

## 컴포넌트 분리 패턴

### 패턴 1: 레이어 분리

```
application-layer/
├── usecase/
├── command/
└── result/

domain-layer/
├── entity/
├── value/
└── service/

infrastructure-layer/
├── persistence/
├── client/
└── event/

적용 시기:
- 명확한 레이어 구조
- 기술 독립성 필요
- 테스트 용이성 중요
```

### 패턴 2: 도메인 분리

```
order-component/
├── order-domain/
├── order-application/
└── order-infrastructure/

payment-component/
├── payment-domain/
├── payment-application/
└── payment-infrastructure/

적용 시기:
- 도메인이 명확히 구분됨
- 독립적인 배포 필요
- 팀 단위 분리
```

### 패턴 3: 기능 분리

```
order-creation/
├── CreateOrderUseCase
├── OrderValidator
└── OrderFactory

order-cancellation/
├── CancelOrderUseCase
├── CancellationPolicy
└── RefundCalculator

적용 시기:
- 기능이 복잡함
- 독립적인 변경 빈도
- 재사용성 낮음
```

### 패턴 4: 공통 추출

```
common-value/
├── Money
├── Email
└── Phone

common-event/
└── DomainEvent

common-util/
├── DateUtil
└── StringUtil

적용 시기:
- 여러 곳에서 재사용
- 매우 안정적
- 독립적인 릴리스
```

---

## 안티패턴과 해결책

### 안티패턴 1: God Component

**문제**:

```
❌ common/
├── Money
├── Email
├── Order
├── User
├── Payment
├── StringUtils
├── DateUtils
└── ... (100+ 클래스)

→ 모든 것을 담음
→ 변경 영향 파악 불가
→ 재사용 어려움
```

**해결**:

```
✅ 주제별로 분리

common-value/
├── Money
├── Email
└── Phone

common-util/
├── StringUtils
└── DateUtils

order-domain/
└── Order

user-domain/
└── User
```

### 안티패턴 2: 순환 의존성

**문제**:

```
❌ order-domain
      ↓
  payment-domain
      ↓
  voucher-domain
      ↓
  order-domain (순환!)
```

**해결책 1: 이벤트 분리**:

```
✅ order-domain
      ↓
  domain-event
      ↑
  payment-domain
```

**해결책 2: 인터페이스 분리**:

```
✅ order-domain
      ↓
  order-port (interface)
      ↑
  payment-domain (구현)
```

### 안티패턴 3: 잘못된 추상화

**문제**:

```
❌ common-abstract/
├── AbstractService
├── AbstractRepository
├── AbstractValidator
└── AbstractPolicy

→ 구체적인 것이 없는 추상화
→ 사용되지 않음
→ "쓸모없는 구역"
```

**해결**:

```
✅ 필요할 때만 추상화

domain-port/
├── OrderRepository (필요함)
└── PaymentProcessor (필요함)

→ 구체적인 구현체가 존재
→ 실제로 사용됨
```

### 안티패턴 4: 과도한 분리

**문제**:

```
❌ order-id/          # OrderId만
order-status/     # OrderStatus만
order-item/       # OrderItem만
order-entity/     # Order만

→ 너무 잘게 쪼갬
→ 관리 비용 증가
→ 의존성 복잡
```

**해결**:

```
✅ 적절한 크기로 응집

order-domain/
├── Order
├── OrderId
├── OrderStatus
└── OrderItem

→ 함께 사용됨
→ 함께 변경됨
→ 관리 용이
```

### 안티패턴 5: 기술 기반 분리

**문제**:

```
❌ controllers/    # 모든 Controller
services/      # 모든 Service
repositories/  # 모든 Repository

→ 기술로만 분리
→ 비즈니스 응집도 낮음
→ 변경 영향 파악 어려움
```

**해결**:

```
✅ 도메인 기반 분리

order/
├── OrderController
├── OrderService
└── OrderRepository

payment/
├── PaymentController
├── PaymentService
└── PaymentRepository

→ 도메인 응집도 높음
→ 변경 영향 명확
```

---

## 측정과 개선

### 컴포넌트 메트릭 측정

**측정 항목**:

```
1. 불안정성 (I)
I = Fan-out / (Fan-in + Fan-out)

2. 추상화 (A)
A = 추상 클래스 수 / 전체 클래스 수

3. 주계열로부터의 거리 (D)
D = |A + I - 1|
(0에 가까울수록 이상적)

4. 순환 의존성
- 순환 횟수
- 순환 길이
```

**개선 전략**:

```
1. D > 0.5 (주계열에서 멀리 떨어짐)
   → 구조 개선 필요

2. 순환 의존성 존재
   → 즉시 제거

3. I < 0.2 && A < 0.3 (고통의 구역)
   → 추상화 도입

4. I > 0.8 && A > 0.7 (쓸모없는 구역)
   → 제거 또는 구체화
```

---

## 체크리스트

### 새 컴포넌트 생성 시

- [ ] REP: 함께 릴리스되는 클래스들인가?
- [ ] CCP: 함께 변경되는 클래스들인가?
- [ ] CRP: 함께 재사용되는 클래스들인가?
- [ ] ADP: 순환 의존성이 없는가?
- [ ] SDP: 안정적인 방향으로 의존하는가?
- [ ] SAP: 안정성에 맞는 추상화 수준인가?

### 컴포넌트 분리 시

- [ ] 분리 이유가 명확한가?
- [ ] 독립적으로 변경 가능한가?
- [ ] 독립적으로 배포 가능한가?
- [ ] 순환 의존성이 생기지 않는가?

### 의존성 추가 시

- [ ] 꼭 필요한 의존성인가?
- [ ] 순환이 생기지 않는가?
- [ ] 안정성 방향이 맞는가?
- [ ] 인터페이스를 통한 의존인가?

---

이 가이드는 컴포넌트 설계와 의존성 관리를 위한 실용적인 원칙을 제공합니다. 프로젝트의 규모와 특성에 맞게 조정하여 사용하시기 바랍니다.
