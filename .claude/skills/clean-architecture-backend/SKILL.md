---
name: clean-architecture-backend
description: Automated architectural decision-making and code implementation based on Clean Architecture, DDD, and object-oriented design. Use when implementing new features ("XX 기능 구현해줘"), reviewing/refactoring code ("이 코드 검토해줘", "리팩토링해줘"), or answering architecture questions ("어느 레이어에 배치해야 해?", "의존성이 맞나?"). Automatically determines layer placement, dependency direction, component boundaries, and responsibility distribution for Java/Kotlin and TypeScript/Node.js projects.
---

# Clean Architecture Backend Developer

Automates architectural decisions and implements backend code following Clean Architecture, Domain-Driven Design, and Responsibility-Driven Design principles.

## Overview

This skill extends beyond simple code generation by actively making architectural decisions:
- **Layer Placement**: Automatically determines which layer code belongs to
- **Dependency Direction**: Ensures dependencies flow toward stability
- **Component Boundaries**: Suggests module and package structures
- **Responsibility Distribution**: Assigns responsibilities following GRASP principles

## Quick Decision Framework

When you receive a coding request, automatically apply this decision tree:

### 1. Determine Layer

**Question: "Why would this code change?"**

```
Business rule change → Domain Layer
Business flow change → Application Layer
External system change → Infrastructure Layer
UI/protocol change → Presentation Layer
```

### 2. Check Dependency Direction

**Rule: Dependencies always point toward Domain**

```
Presentation → Application → Domain ← Infrastructure
                                ↑
                          (never depends on)
```

### 3. Assign Responsibility

**Ask: "Who has the information needed?"**

```
Has the data → Information Expert principle
Creates objects → Creator principle
Multiple actors → Controller (Use Case)
Needs variation → Polymorphism
```

## Workflow

### When Implementing New Features

1. **Analyze Request**
   - Extract domain concepts (nouns → entities/value objects)
   - Extract actions (verbs → methods)
   - Identify aggregates and boundaries

2. **Make Architectural Decisions**
   - Apply decision framework (see above)
   - Determine layer placement automatically
   - Design responsibility distribution
   - Plan dependency structure

3. **Implement Code**
   - Start with Domain Layer (entities, value objects)
   - Then Application Layer (use cases, ports)
   - Finally Infrastructure Layer (adapters)
   - Follow patterns from reference guides

4. **Validate Architecture**
   - Check dependency direction
   - Verify SRP compliance
   - Ensure low coupling, high cohesion
   - Review against checklist

### When Reviewing/Refactoring Code

1. **Identify Issues**
   - Layer violations
   - Wrong dependency direction
   - SRP violations
   - Responsibility misplacement

2. **Suggest Improvements**
   - Correct layer placement
   - Introduce ports/adapters
   - Extract responsibilities
   - Apply design patterns

3. **Implement Refactoring**
   - Preserve behavior (tests first)
   - Refactor incrementally
   - Validate at each step

### When Answering Design Questions

1. **Apply Decision Framework**
   - Use ARCHITECTURE_DECISION_FRAMEWORK.md
   - Consider RESPONSIBILITY_DRIVEN_DESIGN.md principles
   - Check DEPENDENCY_MANAGEMENT.md rules

2. **Provide Concrete Examples**
   - Show before/after code
   - Explain reasoning
   - Highlight trade-offs

## Reference Guides

### Core Architectural Decisions

**[ARCHITECTURE_DECISION_FRAMEWORK.md](references/ARCHITECTURE_DECISION_FRAMEWORK.md)**

Automated decision-making framework including:
- Layer placement decision tree
- Dependency direction verification algorithm
- Component boundary suggestions
- Responsibility assignment patterns
- Real-world decision examples

**Use when**: Making any architectural decision about layer, dependency, or component structure.

### Object-Oriented Design

**[RESPONSIBILITY_DRIVEN_DESIGN.md](references/RESPONSIBILITY_DRIVEN_DESIGN.md)**

Responsibility and collaboration patterns:
- GRASP principles (Information Expert, Creator, Controller, Low Coupling, High Cohesion)
- Responsibility assignment process
- Collaboration design
- Anti-patterns to avoid

**Use when**: Determining which class should have which responsibility or designing object interactions.

### Dependency Management

**[DEPENDENCY_MANAGEMENT.md](references/DEPENDENCY_MANAGEMENT.md)**

Comprehensive dependency control:
- Dependency Rule enforcement
- Dependency Inversion Principle (DIP)
- Metrics (Afferent/Efferent Coupling, Instability, Abstractness)
- Plugin architecture patterns
- Circular dependency resolution

**Use when**: Managing dependencies, applying DIP, or resolving dependency issues.

### Code Quality

**[CODE_REVIEW_CHECKLIST.md](references/CODE_REVIEW_CHECKLIST.md)**

Systematic code review guide:
- Architecture compliance checks
- Domain model validation
- Responsibility and collaboration review
- Use Case structure verification
- Infrastructure layer checks
- Code quality standards

**Use when**: Reviewing code or validating implementation against architectural principles.

### Domain Design (from writing-backend-code)

**[DOMAIN_DESIGN_GUIDE.md](../writing-backend-code/references/DOMAIN_DESIGN_GUIDE.md)**

Detailed domain modeling patterns:
- Entity design patterns
- Value Object implementation
- Aggregate boundaries
- Domain Service patterns
- Domain Event patterns

**Use when**: Implementing domain models or need detailed DDD patterns.

## Automatic Decision Examples

### Example 1: Feature Request

**User**: "사용자가 주문을 취소할 수 있게 해주세요"

**Automated Analysis**:
```
1. Domain Concepts:
   - "주문" (Order) - Entity
   - "취소" (cancel) - Domain method

2. Layer Decision:
   - "주문 취소" → Business rule → Domain Layer
   - Cancel flow control → Application Layer

3. Responsibility:
   - Cancellation validation → Order (Information Expert)
   - Flow orchestration → CancelOrderUseCase (Controller)
   - Refund processing → PaymentGateway (Port)

4. Implementation Plan:
   - Domain: Order.cancel() method
   - Application: CancelOrderUseCase class
   - Infrastructure: Port interfaces only
```

**Generated Code**:

```java
// Domain Layer
package com.shop.order.domain;

public class Order {
    private OrderId id;
    private OrderStatus status;
    private Money totalAmount;

    public void cancel() {
        validateCanCancel();
        this.status = OrderStatus.CANCELLED;
        this.cancelledAt = LocalDateTime.now();
        registerEvent(new OrderCancelledEvent(this.id));
    }

    private void validateCanCancel() {
        if (this.status == OrderStatus.SHIPPED) {
            throw new CannotCancelShippedException(
                "배송 시작된 주문은 취소할 수 없습니다"
            );
        }
        if (this.status == OrderStatus.CANCELLED) {
            throw new OrderAlreadyCancelledException();
        }
    }
}

// Application Layer
package com.shop.order.application;

public class CancelOrderUseCase {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final EventPublisher eventPublisher;

    public CancelOrderUseCase(
            OrderRepository orderRepository,
            PaymentGateway paymentGateway,
            EventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public CancelOrderResult execute(CancelOrderCommand command) {
        // 1. 조회
        Order order = orderRepository.findById(command.getOrderId());

        // 2. 도메인 로직 실행
        order.cancel();

        // 3. 환불 처리
        if (order.isPaid()) {
            paymentGateway.refund(order.getPaymentId());
        }

        // 4. 저장
        orderRepository.save(order);

        // 5. 이벤트 발행
        order.getDomainEvents().forEach(eventPublisher::publish);

        return CancelOrderResult.success();
    }
}

// Application Layer - Port
package com.shop.order.application.port;

public interface PaymentGateway {
    void refund(PaymentId paymentId);
}
```

### Example 2: Code Review

**User**: "이 코드 리뷰해줘"

```java
class OrderService {
    public void processOrder(OrderRequest request) {
        Order order = new Order();
        order.setUserId(request.getUserId());
        order.setItems(request.getItems());

        // 결제 처리
        StripeClient stripe = new StripeClient();
        stripe.charge(request.getCardNumber(), order.getTotal());

        // 저장
        orderRepository.save(order);

        // 이메일 발송
        EmailClient email = new EmailClient();
        email.send(request.getEmail(), "주문 완료");
    }
}
```

**Automated Analysis**:
```
Issues Detected:

1. Layer Violation (P0):
   ❌ Business logic + Infrastructure mixed
   → Separate into Domain, Application, Infrastructure

2. Dependency Direction (P0):
   ❌ Depends on concrete classes (StripeClient, EmailClient)
   → Use interfaces (PaymentGateway, NotificationService)

3. SRP Violation (P1):
   ❌ Handles order creation, payment, notification
   → Split into focused use cases

4. Anemic Domain Model (P1):
   ❌ Order is just a data container
   → Add business methods to Order
```

**Refactored Code**:

```java
// Domain Layer
package com.shop.order.domain;

public class Order {
    private final OrderId id;
    private UserId userId;
    private List<OrderItem> items;
    private OrderStatus status;

    public static Order create(UserId userId, List<OrderItem> items) {
        Order order = new Order();
        order.id = OrderId.generate();
        order.userId = userId;
        order.items = new ArrayList<>(items);
        order.status = OrderStatus.PENDING;
        order.validate();
        return order;
    }

    private void validate() {
        if (items.isEmpty()) {
            throw new EmptyOrderException();
        }
    }

    public Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// Application Layer
package com.shop.order.application;

public class ProcessOrderUseCase {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final NotificationService notificationService;

    public ProcessOrderUseCase(
            OrderRepository orderRepository,
            PaymentGateway paymentGateway,
            NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.notificationService = notificationService;
    }

    @Transactional
    public ProcessOrderResult execute(ProcessOrderCommand command) {
        // 1. 주문 생성
        Order order = Order.create(
            command.getUserId(),
            command.getItems()
        );

        // 2. 결제 처리
        PaymentResult payment = paymentGateway.charge(
            command.getPaymentMethod(),
            order.calculateTotal()
        );

        // 3. 저장
        orderRepository.save(order);

        // 4. 알림
        notificationService.notifyOrderPlaced(order.getId());

        return ProcessOrderResult.success(order.getId());
    }
}

// Application Layer - Ports
package com.shop.order.application.port;

public interface PaymentGateway {
    PaymentResult charge(PaymentMethod method, Money amount);
}

public interface NotificationService {
    void notifyOrderPlaced(OrderId orderId);
}

// Infrastructure Layer - Adapters
package com.shop.order.infrastructure;

public class StripePaymentAdapter implements PaymentGateway {
    private final StripeClient stripeClient;

    @Override
    public PaymentResult charge(PaymentMethod method, Money amount) {
        // Stripe API 호출 및 결과 변환
    }
}

public class EmailNotificationAdapter implements NotificationService {
    private final EmailClient emailClient;

    @Override
    public void notifyOrderPlaced(OrderId orderId) {
        // 이메일 발송 로직
    }
}
```

## Decision Automation Rules

### Rule 1: If Contains Business Validation → Domain

```java
// Detected pattern: validation, business rules
if (code.contains("validate") ||
    code.contains("business rule") ||
    code.contains("invariant")) {
    layer = "Domain";
}
```

### Rule 2: If Depends on Framework → Infrastructure

```java
// Detected pattern: framework annotations, external libraries
if (code.hasAnnotation("@Entity") ||
    code.hasAnnotation("@Repository") ||
    code.imports("javax.persistence")) {
    layer = "Infrastructure";
}
```

### Rule 3: If Orchestrates Flow → Application

```java
// Detected pattern: workflow, multiple service calls
if (code.calls(multipleServices) &&
    code.hasTransactionBoundary()) {
    layer = "Application";
    pattern = "Use Case";
}
```

### Rule 4: If Direction Wrong → Insert Interface

```java
// Detected: wrong dependency direction
if (application.dependsOn(infrastructure)) {
    solution = "Create Port interface in Application";
    action = "Make Infrastructure implement Port";
}
```

## Implementation Principles

### Always Follow

1. **Domain First**
   - Start with domain model
   - Pure business logic only
   - No framework dependencies

2. **Ports Before Adapters**
   - Define interfaces in Application
   - Implement in Infrastructure
   - Never reverse

3. **Tell, Don't Ask**
   - Objects do their own work
   - Avoid getter chains
   - Command over query

4. **Small and Focused**
   - One class, one responsibility
   - Methods < 20 lines
   - Classes < 300 lines

### Never Do

1. **Domain Pollution**
   - No @Entity in domain
   - No JPA in domain
   - No framework in domain

2. **Wrong Dependencies**
   - Domain → Infrastructure (never!)
   - Application → Implementation (use Port!)
   - Circular dependencies (break with events!)

3. **Anemic Models**
   - Getters/setters only
   - Logic in services
   - Data without behavior

4. **God Objects**
   - Too many responsibilities
   - Too many dependencies
   - Too many lines

## Output Guidelines

### When Implementing

1. **Show Decision Process**
   - Why this layer?
   - Why this responsibility?
   - What alternatives considered?

2. **Generate Complete Code**
   - All layers involved
   - Proper package structure
   - Test examples

3. **Explain Trade-offs**
   - Design choices
   - Performance implications
   - Maintenance considerations

### When Reviewing

1. **Identify Issues by Priority**
   - P0: Architecture violations
   - P1: Design principles
   - P2: Code quality
   - P3: Style

2. **Provide Concrete Fixes**
   - Show refactored code
   - Explain improvements
   - Reference principles

3. **Prevent Future Issues**
   - Suggest patterns
   - Add validation
   - Improve tests

## Language-Specific Patterns

### Java/Kotlin + Spring

```java
// Domain - Pure Java/Kotlin
class Order {
    // No Spring, no JPA
}

// Application - Spring stereotypes OK
@UseCase  // or @Service
class PlaceOrderUseCase {
    // Port interfaces
}

// Infrastructure - Spring + JPA
@Repository
class OrderJpaAdapter implements OrderRepository {
    // JPA entities separate from domain
}
```

### TypeScript/Node.js + NestJS

```typescript
// Domain - Pure TypeScript
export class Order {
    // No NestJS decorators
}

// Application - NestJS OK
@Injectable()
export class PlaceOrderUseCase {
    // Port interfaces
}

// Infrastructure - NestJS + TypeORM
@Injectable()
export class OrderTypeOrmAdapter implements OrderRepository {
    // TypeORM entities separate from domain
}
```

## Success Metrics

Your implementation is successful when:

- [ ] Dependencies point toward Domain
- [ ] Each class has one clear responsibility
- [ ] Domain is framework-agnostic
- [ ] Tests are simple and fast
- [ ] Code reads like business language
- [ ] Changes are localized to one layer
- [ ] New features are easy to add
- [ ] Refactoring is low-risk

## Remember

**This skill actively makes decisions for you.**

You don't just generate code—you:
1. Analyze the request
2. Make architectural decisions
3. Apply design principles
4. Generate quality code
5. Validate against standards
6. Explain your reasoning

**Think like an architect, code like a craftsman.**
