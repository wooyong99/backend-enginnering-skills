---
name: architecture-design
description: Clean architecture-based design document generator that must run before feature implementation. Use when designing new features, defining domain boundaries, or determining dependency directions. Outputs architecture design md files that minimize cost of change by applying SOLID and component principles. Purpose is design document generation, not code generation.
---

# Architecture Design Document Generator

Generate architecture design documents before feature implementation. Output is md file, not code.

## Core Principles

### Software Value

1. **Behavior**: Functionality that satisfies requirements
2. **Structure**: Flexibility to accommodate change

**Structure is more important than behavior.** Minimizing cost of change is the top priority.

### Dependency Rule

```
[High-level Policy] ──→ [Abstraction] ←── [Low-level Detail]
                              ↑
              Dependencies always point inward
```

- High-level policies (use cases, business rules) must not directly depend on low-level details (DB, frameworks, external APIs)
- Same applies between domains: upper domains must not depend on lower domain implementations

### Cohesion-First Rule

**Low coupling without cohesion is not allowed.**

1. First, ensure cohesion (group things that change together)
2. Then, reduce coupling (block change propagation)

## Workflow

### Step 1: Requirements Analysis

Collect functional requirements from user and classify:

**Functional Requirements**
- Core use cases
- Input/output specifications
- Business rules

**Non-Functional Requirements**
- Performance (response time, throughput)
- Fault tolerance (availability, recovery)
- Security (authentication, authorization, encryption)
- Observability (logging, metrics, tracing)
- Operations (deployment, scalability)

### Step 2: Identify Actors and Reasons for Change

For each requirement:
- **Who (Actor)** requests this functionality?
- **Why (Reason for Change)** might this functionality change?

Group items with same reason for change → Basis for component boundaries

### Step 3: Design Domain Boundaries

1. Identify core and supporting domains
2. Determine policy level between domains (upper/lower)
3. Select boundary mechanism:
   - **Abstraction (Interface)**: Simple dependency inversion
   - **Ports/Adapters**: External system isolation
   - **Domain Events**: Loose coupling, async communication
   - **ACL (Anti-Corruption Layer)**: Prevent external model pollution

Detailed guide: [references/boundaries.md](references/boundaries.md)

### Step 4: Verify Component Principles

**Cohesion Principles** (What to group together)
- REP: Reuse unit = Release unit
- CCP: Group things that change together
- CRP: Don't depend on things not used together

**Coupling Principles** (How to relate components)
- ADP: No cyclic dependencies
- SDP: Depend in direction of stability
- SAP: Stable components should be abstract

Detailed guide: [references/principles.md](references/principles.md)

### Step 5: Design Consistency and Reliability

**Transaction Boundaries**
- Within single aggregate: Strong consistency
- Between aggregates: Eventual consistency

**Reliability Strategies**
- Idempotency: Safe repeated processing of same request
- Retry: Recovery from transient failures
- Compensating transactions: Rollback strategy on failure

### Step 6: Define Test Strategy

| Boundary | Test Type | Purpose |
|----------|-----------|---------|
| Domain internal | Unit tests | Verify business rules |
| Ports/Adapters | Integration tests | Verify external system integration |
| Between services | Contract tests | Verify API compatibility |

### Step 7: Design Observability

- **Structured logs**: Searchable JSON format
- **Trace ID**: Request flow tracking
- **Key metrics**: Latency, error rate, throughput
- **Alert points**: SLO violation detection

### Step 8: Document Design Decisions

For each major decision:
1. Decision made
2. Alternatives considered
3. Rationale (trade-offs)
4. Expected impact

## Output

Generate architecture design document using [assets/template.md](assets/template.md) template.

Complete all sections without omission, and specify the rationale for each boundary from the perspective of reasons for change.
