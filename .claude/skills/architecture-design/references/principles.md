# Design Principles Reference

## Table of Contents

1. [SOLID Principles](#solid-principles)
2. [Component Cohesion Principles](#component-cohesion-principles)
3. [Component Coupling Principles](#component-coupling-principles)
4. [Cohesion-First Rule](#cohesion-first-rule)

---

## SOLID Principles

### SRP (Single Responsibility Principle)

**A module should be responsible to one, and only one, actor.**

```
# Bad: Multiple actors depend on one class
Employee
├── calculatePay()    → CFO (Finance)
├── reportHours()     → COO (Operations)
└── save()            → CTO (Technology)

# Good: Separated by actor
PayrollCalculator     → CFO
HourReporter          → COO
EmployeeRepository    → CTO
```

**Validation question**: "Is there only one reason for this module to change?"

### OCP (Open-Closed Principle)

**Open for extension, closed for modification.**

New functionality should be added without modifying existing code.

```
# Bad: Adding new type requires modifying existing code
def calculate_area(shape):
    if shape.type == "circle":
        return 3.14 * shape.radius ** 2
    elif shape.type == "rectangle":  # Modified for each new type
        return shape.width * shape.height

# Good: Extension through abstraction
class Shape(ABC):
    @abstractmethod
    def area(self) -> float: pass

class Circle(Shape):
    def area(self) -> float:
        return 3.14 * self.radius ** 2
```

### LSP (Liskov Substitution Principle)

**Subtypes must be substitutable for their base types.**

In inheritance, child must not violate parent's contract.

```
# Bad: Square violates Rectangle contract
class Rectangle:
    def set_width(self, w): self.width = w
    def set_height(self, h): self.height = h

class Square(Rectangle):  # Violation: setting width also changes height
    def set_width(self, w):
        self.width = w
        self.height = w  # Unexpected behavior
```

### ISP (Interface Segregation Principle)

**Clients should not depend on methods they do not use.**

```
# Bad: Unnecessary dependencies
class Worker(ABC):
    @abstractmethod
    def work(self): pass
    @abstractmethod
    def eat(self): pass  # Robots don't eat

# Good: Segregated interfaces
class Workable(ABC):
    @abstractmethod
    def work(self): pass

class Eatable(ABC):
    @abstractmethod
    def eat(self): pass
```

### DIP (Dependency Inversion Principle)

**High-level modules should not depend on low-level modules. Both should depend on abstractions.**

```
# Bad: High-level → Low-level direct dependency
class OrderService:
    def __init__(self):
        self.repository = MySQLOrderRepository()  # Concrete type dependency

# Good: Depend on abstraction
class OrderService:
    def __init__(self, repository: OrderRepository):  # Interface dependency
        self.repository = repository
```

---

## Component Cohesion Principles

### REP (Reuse/Release Equivalence Principle)

**The granule of reuse is the granule of release.**

- Reusable components must have release numbers
- All classes in component are released together
- To reuse one, you must take all

**Validation**: "Is it natural for all classes in this component to be released together?"

### CCP (Common Closure Principle)

**Gather together those things that change for the same reasons and at the same times.**

- Component version of SRP
- When change occurs, only one component should need modification

```
# Bad: Different reasons for change grouped together
payment/
├── PaymentProcessor.java    # Payment policy change
├── PaymentRepository.java   # DB schema change
└── PaymentEmailSender.java  # Email format change

# Good: Separated by reason for change
payment-domain/
└── PaymentProcessor.java    # Business rules
payment-infra/
├── PaymentRepository.java   # Infrastructure
└── PaymentEmailSender.java  # External integration
```

**Validation**: "Do classes in this component change for the same reason?"

### CRP (Common Reuse Principle)

**Don't force users to depend on things they don't use.**

- Depending on component means depending on all its classes
- Affected by changes in unused classes

```
# Bad: Partially used component
utils/
├── StringUtils.java      # Frequently used
├── DateUtils.java        # Frequently used
├── PdfGenerator.java     # Occasionally used (heavy dependencies)
└── ImageProcessor.java   # Rarely used

# Good: Separated by usage pattern
common-utils/
├── StringUtils.java
└── DateUtils.java
document-utils/
└── PdfGenerator.java
```

**Validation**: "Will depending on this component bring in unused things?"

---

## Component Coupling Principles

### ADP (Acyclic Dependencies Principle)

**There must be no cycles in the component dependency graph.**

```
# Bad: Cyclic dependency
A → B → C → A  (Cycle!)

# Solution 1: Dependency inversion with DIP
A → B → C
↑       ↓
└── Interface ←┘

# Solution 2: Extract common into new component
    D (Common)
   ↗ ↖
  A   C
   ↘ ↗
    B
```

### SDP (Stable Dependencies Principle)

**Depend in the direction of stability.**

Stability = Incoming dependencies / (Incoming + Outgoing dependencies)

```
# Stable (hard to change): Many things depend on it
        ┌── A
Domain ←┼── B
        └── C

# Unstable (easy to change): Depends on other things
Adapter ──→ Domain
        ──→ External
```

**Rule**: Unstable components should depend on stable components

### SAP (Stable Abstractions Principle)

**Stable components should be abstract, unstable components should be concrete.**

```
Abstractness = Abstract classes & interfaces / Total classes
```

| Stability | Abstractness | Description |
|-----------|--------------|-------------|
| High | High | ✓ Domain core (interface-centric) |
| High | Low | ✗ Zone of Pain (hard to change) |
| Low | Low | ✓ Implementations, adapters |
| Low | High | ✗ Zone of Uselessness |

---

## Cohesion-First Rule

### Principle

**Low coupling without cohesion is not allowed.**

```
# Wrong order
1. Separate to reduce coupling
2. Leave cohesion broken
→ Must modify multiple places for single change

# Correct order
1. First ensure cohesion (CCP: group things that change together)
2. Then minimize coupling (interfaces, events, etc.)
```

### Validation Checklist

Cohesion check:
- [ ] Is code that changes for the same reason located together?
- [ ] Is business concept not scattered across multiple modules?
- [ ] Does adding a feature require modifying only one component?

Coupling check (after ensuring cohesion):
- [ ] Are component dependencies connected through abstractions?
- [ ] Are there no cyclic dependencies?
- [ ] Is change propagation blocked at boundaries?

### Failure Patterns

```
# Anti-pattern: Distributed Monolith
- Services are separated but
- Single feature change requires deploying multiple services
- Cause: Reduced coupling without ensuring cohesion

# Solution: Redefine boundaries
- Reset boundaries based on reason for change (policy/actor)
- Group things that change together into single deployment unit
```
