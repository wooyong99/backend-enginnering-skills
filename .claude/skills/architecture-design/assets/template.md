# [Feature Name] Architecture Design Document

> Date: YYYY-MM-DD
> Version: 1.0
> Status: Draft | Review | Approved

---

## 1. Overview

### 1.1 Purpose

[Describe the problem this feature solves and its business value]

### 1.2 Scope

[Specify what this design document covers and does not cover]

---

## 2. Requirements

### 2.1 Functional Requirements

| ID | Requirement | Priority | Notes |
|----|-------------|----------|-------|
| FR-01 | [Core use case 1] | Must | |
| FR-02 | [Core use case 2] | Must | |
| FR-03 | [Additional feature] | Should | |

#### 2.1.1 Use Case Details

**UC-01: [Use Case Name]**
- Actor: [Who uses this feature]
- Preconditions: [Required state before execution]
- Main Flow:
  1. [Step 1]
  2. [Step 2]
  3. [Step 3]
- Alternative Flow: [Exception handling]
- Postconditions: [State after execution]

#### 2.1.2 Business Rules

| ID | Rule | Rationale |
|----|------|-----------|
| BR-01 | [Business rule 1] | [Policy rationale] |
| BR-02 | [Business rule 2] | [Policy rationale] |

### 2.2 Non-Functional Requirements

#### 2.2.1 Performance

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| Response time (p99) | < [X]ms | [Tool/method] |
| Throughput | [X] TPS | [Tool/method] |
| Concurrent users | [X] | [Tool/method] |

#### 2.2.2 Fault Tolerance

| Item | Requirement |
|------|-------------|
| Availability | [X]% (monthly downtime < [Y] minutes) |
| RTO (Recovery Time Objective) | [X] minutes |
| RPO (Recovery Point Objective) | [X] minutes |
| Fault isolation | [Isolation strategy] |

#### 2.2.3 Security

| Item | Requirement |
|------|-------------|
| Authentication | [Auth method] |
| Authorization | [Permission model] |
| Data encryption | In transit: [method], At rest: [method] |
| Audit logging | [Logging targets and retention period] |

#### 2.2.4 Observability

| Item | Requirement |
|------|-------------|
| Log level | [Logging policy] |
| Metrics | [Collection targets] |
| Tracing | [Distributed tracing approach] |
| Alerts | [Alert conditions and response] |

#### 2.2.5 Operations

| Item | Requirement |
|------|-------------|
| Deployment strategy | [Blue-Green / Canary / Rolling] |
| Scalability | [Horizontal/vertical scaling strategy] |
| Backup | [Backup frequency and retention] |

---

## 3. Boundary Design

### 3.1 Actor and Reason for Change Analysis

| Actor | Concern | Reason for Change | Affected Components |
|-------|---------|-------------------|---------------------|
| [Actor1] | [Concern] | [Change trigger] | [Component list] |
| [Actor2] | [Concern] | [Change trigger] | [Component list] |

### 3.2 Domain Boundaries

```
[Domain Boundary Diagram]

┌─────────────────────────────────────────────────┐
│                 Upper Domain                     │
│  ┌─────────────┐                               │
│  │  Core       │ ← Core business rules          │
│  │  Domain     │                               │
│  └──────┬──────┘                               │
│         │ (Abstraction)                         │
│         ↓                                      │
│  ┌─────────────┐    ┌─────────────┐           │
│  │  Support    │    │  Support    │           │
│  │  Domain A   │    │  Domain B   │           │
│  └─────────────┘    └─────────────┘           │
└─────────────────────────────────────────────────┘
```

### 3.3 Boundary Rationale

| Boundary | Internal Responsibility | Reason for Change | Mechanism |
|----------|------------------------|-------------------|-----------|
| [Boundary1] | [Scope of responsibility] | [Policy/actor based rationale] | [Interface/Event/ACL] |
| [Boundary2] | [Scope of responsibility] | [Policy/actor based rationale] | [Interface/Event/ACL] |

### 3.4 Dependency Direction

```
[Dependency Diagram]

High-level Policy              Low-level Detail
┌──────────┐                  ┌──────────┐
│ UseCase  │ ──→ Port ←────── │ Adapter  │
└──────────┘   (Abstraction)  └──────────┘
     │                              │
     ↓                              ↓
┌──────────┐                  ┌──────────┐
│ Domain   │                  │ Infra    │
│ Entity   │                  │ (DB,API) │
└──────────┘                  └──────────┘
```

---

## 4. Component Design

### 4.1 Component Structure

| Component | Responsibility | Cohesion Principle Validation | Stability |
|-----------|----------------|------------------------------|-----------|
| [Component1] | [Responsibility] | CCP: [Validation] | [High/Low] |
| [Component2] | [Responsibility] | CCP: [Validation] | [High/Low] |

### 4.2 Dependency Validation

- [ ] No cyclic dependencies (ADP)
- [ ] Depend in direction of stability (SDP)
- [ ] Stable components are abstract (SAP)
- [ ] Coupling minimized after cohesion ensured

---

## 5. Consistency and Reliability Design

### 5.1 Transaction Boundaries

| Aggregate | Consistency Model | Boundary Scope |
|-----------|-------------------|----------------|
| [Aggregate1] | Strong consistency | [Included entities] |
| [Aggregate2] | Eventual consistency | [Included entities] |

### 5.2 Inter-Aggregate Communication

| Sender | Receiver | Event/Message | Consistency |
|--------|----------|---------------|-------------|
| [Sender] | [Receiver] | [Event name] | Eventual consistency |

### 5.3 Idempotency Design

| Operation | Idempotency Key | Duplicate Handling Strategy |
|-----------|-----------------|----------------------------|
| [Operation1] | [Key generation method] | [Return cached/Ignore] |
| [Operation2] | [Key generation method] | [Return cached/Ignore] |

### 5.4 Retry Strategy

| Operation | Retry Condition | Backoff | Max Attempts |
|-----------|-----------------|---------|--------------|
| [Operation1] | [Transient failure] | Exponential backoff | [N] times |
| [Operation2] | [Timeout] | Fixed interval | [N] times |

### 5.5 Compensating Transactions (Saga)

```
[Saga Flow Diagram]

Normal flow:
T1: [Transaction1] → T2: [Transaction2] → T3: [Transaction3]

Compensation flow (on T3 failure):
C2: [Compensation2] ← C1: [Compensation1]
```

| Transaction | Operation | Compensating Operation |
|-------------|-----------|----------------------|
| T1 | [Normal operation] | C1: [Rollback operation] |
| T2 | [Normal operation] | C2: [Rollback operation] |

---

## 6. Test Strategy

### 6.1 Test Pyramid

| Level | Scope | Purpose | Tools |
|-------|-------|---------|-------|
| Unit | Domain logic | Verify business rules | [Test framework] |
| Integration | Adapters | Verify external integration | [Test tools] |
| Contract | Inter-service | API compatibility | [Pact/Spring Cloud Contract] |
| E2E | Full flow | Scenario verification | [E2E tools] |

### 6.2 Boundary-specific Test Strategy

| Boundary | Test Type | Test Target | Mock Target |
|----------|-----------|-------------|-------------|
| [Domain] | Unit | [Entities, Services] | None |
| [Port] | Integration | [Adapters] | [External systems] |
| [Inter-service] | Contract | [API spec] | [Consumer/Provider] |

---

## 7. Observability Design

### 7.1 Structured Logs

```json
{
  "timestamp": "ISO8601",
  "level": "INFO|WARN|ERROR",
  "traceId": "[Trace ID]",
  "spanId": "[Span ID]",
  "service": "[Service name]",
  "event": "[Event type]",
  "message": "[Log message]",
  "context": {
    "[Business key]": "[Value]"
  }
}
```

### 7.2 Key Metrics

| Metric | Type | Description | Alert Threshold |
|--------|------|-------------|-----------------|
| [Metric1] | Counter/Gauge/Histogram | [Description] | [Threshold] |
| [Metric2] | Counter/Gauge/Histogram | [Description] | [Threshold] |

### 7.3 Tracing Design

| Span | Trace Points | Collected Information |
|------|--------------|----------------------|
| [Span1] | [Start/End points] | [Context to collect] |
| [Span2] | [Start/End points] | [Context to collect] |

### 7.4 Alert Design

| Alert | Condition | Severity | Response |
|-------|-----------|----------|----------|
| [Alert1] | [Condition expression] | Critical/Warning | [Response procedure] |
| [Alert2] | [Condition expression] | Critical/Warning | [Response procedure] |

---

## 8. Architecture Decision Records (ADR)

### ADR-001: [Decision Title]

**Status**: Proposed | Accepted | Deprecated | Superseded

**Context**: [Background for this decision]

**Decision**: [Chosen option]

**Alternatives**:
1. [Alternative 1]: [Pros and cons]
2. [Alternative 2]: [Pros and cons]
3. [Alternative 3]: [Pros and cons]

**Rationale**: [Trade-off analysis and reason for choice]

**Expected Impact**:
- Positive: [Expected benefits]
- Negative: [Expected costs/constraints]

---

## 9. Implementation Guide

### 9.1 Directory Structure

```
src/
├── domain/           # Core business logic
│   ├── entity/
│   ├── service/
│   └── event/
├── application/      # Use cases
│   ├── port/
│   │   ├── in/      # Inbound ports
│   │   └── out/     # Outbound ports
│   └── service/
├── adapter/          # Infrastructure adapters
│   ├── in/          # Driving adapters
│   │   ├── web/
│   │   └── message/
│   └── out/         # Driven adapters
│       ├── persistence/
│       └── external/
└── config/          # Configuration
```

### 9.2 Dependency Rule Checklist

- [ ] domain package has no external dependencies
- [ ] application depends only on domain
- [ ] adapter depends only on application's ports
- [ ] No cyclic dependencies

---

## 10. Review Checklist

### 10.1 Principle Compliance

- [ ] SRP: Each module has only one reason to change
- [ ] OCP: Open for extension, closed for modification
- [ ] LSP: Subtypes are substitutable for base types
- [ ] ISP: No unnecessary dependencies
- [ ] DIP: High-level does not directly depend on low-level

### 10.2 Component Principles

- [ ] CCP: Things that change together are together
- [ ] CRP: Things not used together are separated
- [ ] ADP: No cyclic dependencies
- [ ] SDP: Depend in direction of stability
- [ ] SAP: Stable components are abstract

### 10.3 Cohesion First

- [ ] Coupling reduced only after cohesion ensured
- [ ] No distributed monolith patterns

---

## Change History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | YYYY-MM-DD | [Author] | Initial creation |
