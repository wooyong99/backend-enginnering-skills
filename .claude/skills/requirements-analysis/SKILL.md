---
name: requirements-analysis
description: Analyze and interpret requirements from a Domain-Driven Design perspective, generating normalized documentation. Use when users request "요구사항 분석해줘", "도메인 모델링해줘", "기획서 검토해줘", "이 기능 설계해줘", or when analyzing specification documents, planning documents, or feature requests that need domain concept extraction and structured analysis before implementation.
context: fork
agent: ddd-architecture-orchestrator
---

# Requirements Analysis Skill

## Overview

Analyze requirements from a Domain-Driven Design (DDD) and Clean Architecture perspective to extract domain concepts, define ubiquitous language, identify bounded contexts, and produce structured analysis documents. This skill transforms ambiguous business requirements into clear, implementation-ready domain models.

## Workflow

### 1. Initial Assessment

When a requirements analysis request arrives, first understand:

**Input Type Identification:**

- Specification document (PDF, Markdown, Word)
- Verbal/chat-based requirements
- Feature request or planning document
- Legacy code analysis request

**Scope Clarification:**

- Which business area does this cover?
- What is the expected level of detail?
- Are there existing domain models to reference?

**Context Gathering:**

- Read any provided documents completely
- Identify key business concepts from the text
- Note ambiguous or unclear requirements

### 2. Bounded Context Identification

Identify and map bounded contexts:

**Questions to Answer:**

- What are the distinct business capabilities?
- Which concepts should be isolated?
- What are the context boundaries?
- How do contexts relate to each other?

**Output:**

- Context map showing relationships
- Responsibility definition for each context
- Integration patterns between contexts (Customer-Supplier, Anti-Corruption Layer, etc.)

Refer to [DDD_PATTERNS.md](references/DDD_PATTERNS.md) for bounded context patterns.

### 3. Ubiquitous Language Definition

Extract and define the domain vocabulary:

**Process:**

1. Extract nouns from business descriptions → Entity/Value Object candidates
2. Extract verbs from business actions → Domain method names
3. Identify state transitions → Entity lifecycle
4. Define policies and rules → Domain Services

**Guidelines:**

- Use business terms, not technical terms
- Map business verbs to domain methods (e.g., "주문하다" → `Order.place()`)
- Avoid ambiguous terms like "process", "handle", "manage"
- Document term definitions clearly

See [UBIQUITOUS_LANGUAGE_GUIDE.md](references/UBIQUITOUS_LANGUAGE_GUIDE.md) for detailed guidance on:

- Extracting terms from business conversations
- Classifying terms into Entity/Value Object/Service
- Creating verb mappings
- Building and maintaining glossaries

### 4. Domain Model Design

Design the core domain model:

#### 4.1 Entity Identification

**Criteria:**

- Has unique identity
- Has lifecycle
- Needs tracking
- State changes over time

**Design:**

- Define identifier type and generation strategy
- List attributes (prefer Value Objects over primitives)
- Define business methods (use business verbs)
- Document state transitions
- Specify invariants

#### 4.2 Value Object Identification

**Criteria:**

- Identified by attributes only
- Immutable
- Replaceable
- Represents measurement, quantity, or description

**Design:**

- Define attributes
- Specify validation rules
- Define business operations (return new instances)
- Implement equals/hashCode semantics

#### 4.3 Aggregate Design

**Principles:**

- Keep aggregates small (prefer single entity)
- Define clear transactional boundaries
- Reference other aggregates by ID only
- Designate Aggregate Root

**Design:**

- Identify Aggregate Root
- List contained entities
- Define consistency boundaries
- Specify external references

#### 4.4 Domain Service Identification

**Use When:**

- Logic spans multiple entities
- Operation doesn't naturally belong to any entity
- Stateless business rules or calculations

**Design:**

- Clear purpose statement
- List dependencies (entities, value objects)
- Define methods with business-meaningful names

Consult [DOMAIN_DESIGN_GUIDE.md](references/DOMAIN_DESIGN_GUIDE.md) (from writing-backend-code skill) for comprehensive guidance on:

- Entity design patterns
- Value Object implementation
- Aggregate boundaries
- Domain Service patterns

### 5. Use Case Analysis

Define business scenarios:

**For Each Use Case:**

- Purpose: What business goal?
- Actor: Who performs it?
- Preconditions: What must be true before?
- Main flow: Step-by-step normal path
- Alternative flows: Valid variations
- Exception flows: Error cases
- Postconditions: Result state
- Business rules: Constraints and policies

**Output Format:**

- Use case table with ID, name, actor, priority
- Detailed flow description for each

### 6. Business Policy Extraction

Identify and document business rules:

**Policy Documentation:**

- Policy name and purpose
- Conditions and outcomes (if X, then Y)
- Examples with input/output
- Special cases and exceptions

**Types:**

- Validation rules
- Calculation algorithms
- Decision policies
- Workflow rules

### 7. Domain Event Identification

Identify significant domain events:

**Criteria:**

- Represents something that happened (past tense)
- Triggers actions in other contexts
- Needs to be recorded
- Has business significance

**Design:**

- Event name (past tense: OrderPlaced, PaymentCompleted)
- When it occurs
- Payload (IDs and essential data only, not entire entities)
- Subscribers and their reactions

### 8. Document Generation

Generate the final analysis document:

**Using the Template:**

Start with the template from [assets/requirements-analysis-template.md](assets/requirements-analysis-template.md) and populate:

1. **Overview section** - Summary of analysis target and business context
2. **Bounded Context section** - Context map and detailed descriptions
3. **Ubiquitous Language section** - Term definitions and verb mappings
4. **Domain Model section** - Entities, Value Objects, Aggregates, Services
5. **Use Cases section** - Use case list and detailed flows
6. **Business Policies section** - Rules and constraints
7. **Domain Events section** - Event definitions
8. **Constraints section** - Technical and business assumptions
9. **Next Steps section** - Validation needs and priorities

**Output Format:**

- Structured Markdown document
- Save to appropriate location (ask user if unclear)
- Follow the template structure for consistency

### 9. Validation and Clarification

After generating the document:

**Validate:**

- Are all terms clearly defined?
- Are business rules complete?
- Are boundaries clear?
- Are there contradictions?

**Identify Gaps:**

- List unclear requirements
- Note missing information
- Highlight assumptions

**Recommend Next Steps:**

- Prioritize implementation (Core > Support > Generic)
- Suggest technical validations
- List items needing stakeholder clarification

## Key Principles

### Domain-First Thinking

Always prioritize domain over technology:

```
✅ Good Flow:
1. Understand business concepts
2. Model domain
3. Define use cases
4. Then consider technology

❌ Bad Flow:
1. Start with database schema
2. Force business into tables
3. Create CRUD services
```

### Business Language First

Use domain expert vocabulary:

```
✅ Business Terms:
place() - 주문하다
approve() - 승인하다
cancel() - 취소하다

❌ Technical Terms:
updateStatus()
changeState()
modifyData()
```

### Explicit Over Implicit

Make business rules explicit in code:

```
✅ Explicit:
validateCanPlace()
calculateDiscountedPrice()

❌ Implicit:
// Hidden logic in setters
setStatus("PLACED")
```

## Reference Materials

This skill provides comprehensive reference documentation:

### [DDD_PATTERNS.md](references/DDD_PATTERNS.md)

Strategic and tactical DDD patterns including:

- Bounded Context patterns and relationships
- Entity, Value Object, Aggregate patterns
- Domain Service and Repository patterns
- Specification pattern
- Anti-patterns to avoid

**Use when:** Designing domain models, identifying patterns, or resolving design questions.

### [UBIQUITOUS_LANGUAGE_GUIDE.md](references/UBIQUITOUS_LANGUAGE_GUIDE.md)

Complete guide to discovering and defining domain language:

- Term extraction process
- Classification guidelines (Entity vs Value Object vs Service)
- Verb mapping patterns
- Glossary templates
- Term conflict resolution

**Use when:** Extracting terms from requirements or building glossaries.

### [ANALYSIS_TEMPLATE.md](references/ANALYSIS_TEMPLATE.md)

Detailed template structure with:

- Section-by-section guidance
- Field descriptions
- Example formats

**Use when:** Needing more detailed guidance than the asset template provides.

### [DOMAIN_DESIGN_GUIDE.md](writing-backend-code/references/DOMAIN_DESIGN_GUIDE.md)

Comprehensive domain modeling guide (from writing-backend-code skill):

- Entity design patterns with code examples
- Value Object implementation
- Aggregate design principles
- State transition management
- Validation strategies
- Domain event patterns

**Use when:** Needing detailed implementation patterns for domain concepts.

## Output Quality Standards

A good requirements analysis document should:

1. **Be Clear and Unambiguous**
   - Every term has a precise definition
   - Business rules are explicit
   - No technical jargon

2. **Be Implementation-Ready**
   - Sufficient detail for developers
   - Domain model is complete
   - Use cases are well-defined

3. **Be Maintainable**
   - Structured consistently
   - Easy to update
   - Version controllable

4. **Reflect Business Reality**
   - Uses domain expert language
   - Captures actual business rules
   - Represents true workflows

## Common Pitfalls to Avoid

1. **Technical Contamination**
   - Don't use database terminology in domain models
   - Avoid framework-specific concepts
   - Focus on business, not implementation

2. **Over-Engineering**
   - Don't create abstractions without clear need
   - Keep aggregates small
   - Avoid premature optimization

3. **Ambiguous Language**
   - Define every important term
   - Don't use synonyms for same concept
   - Avoid words with multiple meanings

4. **Missing Business Rules**
   - Extract all constraints
   - Document validation rules
   - Capture state transition conditions

5. **Weak Boundaries**
   - Clearly define context boundaries
   - Don't mix concerns
   - Respect aggregate boundaries
