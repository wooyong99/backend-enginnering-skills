# Boundary Design Guide

## Table of Contents

1. [Boundary Setting Principles](#boundary-setting-principles)
2. [Boundary Mechanisms](#boundary-mechanisms)
3. [Inter-Domain Dependency Design](#inter-domain-dependency-design)
4. [Consistency and Reliability Patterns](#consistency-and-reliability-patterns)

---

## Boundary Setting Principles

### Reason-for-Change Based Boundaries

Boundaries are set based on **reasons for change (policy/actor)**.

```
Question: "Why would this code need to change?"
          "Who requests this change?"

Same answer → Same boundary
Different answer → Separate boundaries
```

### Boundary Types

| Type | Separation Level | Cost | When to Use |
|------|------------------|------|-------------|
| Source level | Package/Module | Low | Early stage, monolith |
| Deployment level | JAR, DLL | Medium | Independent deployment needed |
| Service level | Process | High | Independent scaling, tech heterogeneity |

**Principle**: Start with the weakest boundary and strengthen when needed.

---

## Boundary Mechanisms

### 1. Abstraction (Interface)

Simplest form of dependency inversion.

```
# Before: Direct dependency
OrderService → MySQLOrderRepository

# After: Inverted with interface
OrderService → OrderRepository (interface)
                     ↑
              MySQLOrderRepository
```

**When to use**: Implementation replacement possibility, testability needed

### 2. Ports/Adapters (Hexagonal)

Isolate external systems from domain.

```
┌─────────────────────────────────────┐
│           Driving Adapters          │
│  (REST Controller, CLI, Message)    │
└──────────────┬──────────────────────┘
               ↓
        ┌──────────────┐
        │  Input Port  │ (Interface)
        └──────┬───────┘
               ↓
        ┌──────────────┐
        │    Domain    │ (Business Logic)
        └──────┬───────┘
               ↓
        ┌──────────────┐
        │ Output Port  │ (Interface)
        └──────┬───────┘
               ↓
┌──────────────────────────────────────┐
│          Driven Adapters             │
│  (DB, External API, Message Queue)   │
└──────────────────────────────────────┘
```

**When to use**: External system integration, technology change possibility

### 3. Domain Events

Loosely coupled async communication.

```
# Synchronous calls (tight coupling)
OrderService.createOrder()
    → InventoryService.reserve()
    → PaymentService.charge()
    → NotificationService.send()

# Event-based (loose coupling)
OrderService.createOrder()
    → publish(OrderCreatedEvent)

InventoryService.on(OrderCreatedEvent) → reserve()
PaymentService.on(OrderCreatedEvent) → charge()
NotificationService.on(OrderCreatedEvent) → send()
```

**When to use**:

- Publisher doesn't need to know subscribers
- Eventual consistency is acceptable
- Independent scaling needed

### 4. ACL (Anti-Corruption Layer)

Prevent external model pollution.

```
┌──────────────┐     ┌─────────────┐     ┌──────────────┐
│ Our Domain   │ ←── │     ACL     │ ←── │ External API │
│              │     │ (Translation│     │ (Legacy etc) │
│ Order        │     │   Layer)    │     │ LegacyOrder  │
│ Customer     │     │ translate() │     │ OldCustomer  │
└──────────────┘     └─────────────┘     └──────────────┘
```

**Includes**:

- External model → Internal model conversion
- External exception → Internal exception conversion
- External protocol encapsulation

**When to use**: Legacy integration, external API dependency, model mismatch

---

## Inter-Domain Dependency Design

### Policy Level Analysis

Classify domains by policy level:

```
Upper policy (Core business)
    ↓ depends on
Lower policy (Supporting functions)
    ↓ depends on
Details (Infrastructure)
```

Example:

```
[Order Domain] - Core policy
      ↓
[Payment Domain] - Supporting policy
      ↓
[Notification Domain] - Details
```

### Dependency Direction Rule

**Upper domains must not depend on lower domain implementations/data models.**

```
# Bad: Upper depends on lower implementation
OrderService
    → PaymentGatewayImpl  # Concrete type
    → PaymentDTO          # Lower domain model

# Good: Inverted with abstraction
OrderService
    → PaymentPort (interface)  # Abstraction
    → OrderPaymentInfo         # Own model

PaymentAdapter implements PaymentPort
    → PaymentGatewayImpl
    → PaymentDTO → OrderPaymentInfo conversion
```

### Boundary Mechanism Selection Guide

| Situation | Recommended Mechanism |
|-----------|----------------------|
| Sync call needed, shared transaction | Interface + DI |
| External system integration | Ports/Adapters |
| Async allowed, eventual consistency | Domain Events |
| Prevent external model pollution | ACL |
| Inter-service communication | Events + ACL combination |

---

## Consistency and Reliability Patterns

### Transaction Boundaries

**Principle**: Set transaction boundaries at Aggregate level

```
# Single Aggregate: Strong consistency
Order (Aggregate Root)
├── OrderItem
└── OrderStatus
→ Atomic change in single transaction

# Between Aggregates: Eventual consistency
Order ──(event)──→ Inventory
→ Separate transactions, eventual consistency
```

### Consistency Model Selection

| Model | Characteristics | When to Use |
|-------|-----------------|-------------|
| Strong consistency | Immediate reflection, sync | Payment, inventory deduction |
| Eventual consistency | Delay allowed, async | Notification, analytics, cache |

### Idempotency Design

Repeated processing of same request must be safe.

```python
# Idempotency key pattern
def process_payment(idempotency_key: str, request: PaymentRequest):
    # 1. Check if request already processed
    existing = find_by_idempotency_key(idempotency_key)
    if existing:
        return existing.result  # Return cached result

    # 2. Process new request
    result = execute_payment(request)

    # 3. Save result (atomic)
    save_with_idempotency_key(idempotency_key, result)
    return result
```

### Retry Strategy

```
# Exponential backoff + jitter
retry_delay = base_delay * (2 ^ attempt) + random_jitter

# Retryable conditions
- Transient failures (timeout, network errors)
- Idempotent operations

# Non-retryable conditions
- Business rule violations (insufficient balance, etc.)
- Authentication/authorization failures
```

### Compensating Transactions (Saga)

Rollback on distributed transaction failure.

```
# Saga pattern example: Order creation

Normal flow:
1. Create order → Success
2. Reserve inventory → Success
3. Process payment → Success

Compensation on failure:
1. Create order → Success
2. Reserve inventory → Success
3. Process payment → Failure!
   → Compensate: Restore inventory
   → Compensate: Cancel order
```

**Implementation approaches**:

- Choreography: Event-based, distributed
- Orchestration: Central coordinator, explicit flow
