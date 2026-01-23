# Backend Engineering Skills

A comprehensive collection of Claude AI skills and agents for building enterprise-grade backend systems following Domain-Driven Design (DDD), Clean Architecture, and Object-Oriented Design (OOD) principles.

## Overview

This project provides an integrated ecosystem of specialized AI agents and skills that work together to guide the entire software development lifecycle—from requirements analysis through implementation to code review. Each component embodies decades of software architecture wisdom, enabling Claude to act as an expert backend architect and developer.

**Current Status:**
- ✅ **Production Ready**: Core DDD architecture orchestration with requirements analysis and automated architectural decision-making
- 🚧 **In Development**: Additional design and task management capabilities

The DDD Architecture Orchestrator agent serves as the strategic coordinator, integrating two production-ready skills (requirements-analysis and clean-architecture-backend) to provide end-to-end architectural guidance.

## Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│         DDD Architecture Orchestrator Agent                  │
│         (Strategic Decision-Making Layer)                    │
└────────────────────┬────────────────────────────────────────┘
                     │
                     │ Integrated Skills
        ┌────────────┴────────────┐
        │                         │
        ▼                         ▼
┌──────────────────┐    ┌──────────────────────┐
│  Requirements    │    │  Clean Architecture  │
│  Analysis        │    │  Backend             │
│  Skill ✅        │    │  Skill ✅            │
└──────────────────┘    └──────────────────────┘


Independent Agents (not integrated with orchestrator):

┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Requirements     │  │ Backend          │  │ Architecture     │
│ Analyst          │  │ Architect        │  │ Reviewer         │
│ Agent            │  │ Agent            │  │ Agent            │
└──────────────────┘  └──────────────────┘  └──────────────────┘


Other Available Skills:

┌──────────────────┐
│  Commit Message  │
│  Skill ✅        │
└──────────────────┘

🚧 Work in Progress Skills:
  • architecture-design
  • spec-to-tasks
  • writing-backend-code
```

## Core Components

### 🎯 DDD Architecture Orchestrator Agent

The **strategic decision-maker** that coordinates all architectural activities. This orchestrator doesn't write code—instead, it makes critical architectural judgments and delegates execution to specialized agents.

**Key Responsibilities:**

- Interprets requirements from a domain-driven perspective
- Makes architectural decisions based on DDD, Clean Architecture, and OOD principles
- Ensures dependency arrows always point toward the domain core
- Coordinates between requirements analysis, design, implementation, and review phases
- Protects domain integrity from technical convenience compromises

**Integrated Skills:**

- ✅ `requirements-analysis` - Production ready
- ✅ `clean-architecture-backend` - Production ready

**When to Use:**

- Starting new feature development
- Encountering ambiguous or complex business requirements
- Making architectural decisions about structure, boundaries, or responsibilities
- Refactoring code with mixed concerns or violated architectural principles
- Resolving conflicts between technical convenience and domain integrity

### 📋 Requirements Analysis Skill ✅

Transforms ambiguous business requirements into clear, implementation-ready domain models using Domain-Driven Design principles.

**Core Capabilities:**

- **Bounded Context Identification**: Maps business capabilities and defines context boundaries
- **Ubiquitous Language Definition**: Extracts domain vocabulary and maps business verbs to domain methods
- **Domain Model Design**: Identifies Entities, Value Objects, Aggregates, and Domain Services
- **Use Case Analysis**: Defines business scenarios with preconditions, flows, and postconditions
- **Business Policy Extraction**: Documents rules, validation, and decision policies
- **Domain Event Identification**: Captures significant business events and their triggers

**Workflow:**

1. Initial Assessment (input type, scope, context gathering)
2. Bounded Context Identification
3. Ubiquitous Language Definition
4. Domain Model Design (Entities, Value Objects, Aggregates, Services)
5. Use Case Analysis
6. Business Policy Extraction
7. Domain Event Identification
8. Document Generation (structured Markdown output)
9. Validation and Clarification

**Reference Materials:**

- `DDD_PATTERNS.md` - Strategic and tactical DDD patterns
- `UBIQUITOUS_LANGUAGE_GUIDE.md` - Term extraction and classification
- `ANALYSIS_TEMPLATE.md` - Detailed template structure
- `requirements-analysis-template.md` - Output template

**Triggers:**

- "요구사항 분석해줘" (Analyze requirements)
- "도메인 모델링해줘" (Model the domain)
- "기획서 검토해줘" (Review the specification)
- "이 기능 설계해줘" (Design this feature)

### 🏗️ Clean Architecture Backend Skill ✅

Automates architectural decision-making and implements backend code following Clean Architecture, Domain-Driven Design, and Responsibility-Driven Design principles.

**Core Capabilities:**

- **Automated Layer Placement**: Determines which architectural layer code belongs to
- **Dependency Direction Enforcement**: Ensures dependencies flow toward stability and domain
- **Component Boundary Definition**: Suggests module and package structures
- **Responsibility Distribution**: Assigns responsibilities following GRASP principles

**Quick Decision Framework:**

```text
Why would this code change?
├─ Business rule change → Domain Layer
├─ Business flow change → Application Layer
├─ External system change → Infrastructure Layer
└─ UI/protocol change → Presentation Layer

Dependency Direction Rule:
Presentation → Application → Domain ← Infrastructure
```

**Automated Decision Rules:**

- If contains business validation → Domain
- If depends on framework → Infrastructure
- If orchestrates flow → Application (Use Case)
- If direction wrong → Insert Port interface

**Reference Materials:**

- `ARCHITECTURE_DECISION_FRAMEWORK.md` - Automated decision algorithms
- `RESPONSIBILITY_DRIVEN_DESIGN.md` - GRASP principles
- `DEPENDENCY_MANAGEMENT.md` - DIP, metrics, circular dependency resolution
- `CODE_REVIEW_CHECKLIST.md` - Systematic review guide

**Triggers:**

- "XX 기능 구현해줘" (Implement XX feature)
- "이 코드 검토해줘" (Review this code)
- "리팩토링해줘" (Refactor this)
- "어느 레이어에 배치해야 해?" (Which layer should this go in?)
- "의존성이 맞나?" (Is the dependency correct?)

**Supported Languages:**

- Java/Kotlin + Spring
- TypeScript/Node.js + NestJS

### 🔍 Backend Architecture Reviewer Agent

Provides architectural guidance and critical thinking about backend design decisions from Clean Architecture, DDD, and object-oriented perspectives.

**Core Focus:**

- Responsibility placement and boundary definition
- Layer separation and dependency direction validation
- Domain pollution detection (framework leakage)
- Refactoring direction validation against architectural principles

**When to Use:**

- Interpreting requirements into domain concepts
- Determining responsibility boundaries between layers
- Reviewing if JPA/ORM/framework code is polluting the domain
- Validating refactoring directions
- Architectural review before implementation

### 📊 Requirements Analyst Agent

Deep analysis of planning documents, requirements specifications, and feature requests before implementation begins.

**Analysis Framework:**

- **Intent Interpretation**: Surface vs. hidden intent, stakeholder perspectives
- **Domain Concept Clarification**: Identify ambiguities and define precise terms
- **Change Rationale**: Understand why changes are needed
- **Impact Assessment**: Evaluate technical and business implications

**When to Use:**

- Receiving new planning documents or requirements specifications
- Feature requests that need interpretation before implementation
- System changes that require understanding the underlying rationale

### 🛠️ Backend Architect Agent

Implements backend architecture based on Clean Architecture principles, creating domain entities, use cases, repository interfaces, controllers, and infrastructure layers.

**Core Expertise:**

- Clean Architecture with clear layer separation
- SOLID principles application
- Domain-Driven Design (bounded contexts, aggregates, entities, value objects)
- Design patterns (Repository, Factory, Strategy, Observer, etc.)

**When to Use:**

- Creating new features following layered architecture patterns
- Structuring backend projects
- Implementing features following SOLID principles and separation of concerns
- Refactoring code to follow Clean Architecture

## Getting Started

### Using the DDD Architecture Orchestrator

The orchestrator is automatically engaged when you:

1. Start new feature development
2. Encounter complex business requirements
3. Make architectural decisions
4. Refactor existing code

Example conversation:

```text
User: "I need to add a payment processing feature to our e-commerce system"

Claude (via Orchestrator): "Let me analyze this requirement from a domain-driven
perspective. I'll identify the core business concepts, define bounded contexts,
and determine the appropriate architectural approach."

→ Orchestrator engages Requirements Analysis Skill
→ Domain model is designed
→ Clean Architecture Backend Skill implements the feature
→ Architecture Reviewer validates the design
```

### Using Requirements Analysis Skill

When you need to analyze business requirements:

```text
User: "요구사항 분석해줘" (Analyze these requirements)
[Attach specification document or describe requirements]

→ Requirements Analysis Skill generates structured documentation:
  - Bounded Contexts
  - Ubiquitous Language
  - Domain Model (Entities, Value Objects, Aggregates)
  - Use Cases
  - Business Policies
  - Domain Events
```

### Using Clean Architecture Backend Skill

When implementing features or refactoring code:

```text
User: "사용자가 주문을 취소할 수 있게 해주세요"
(Enable users to cancel orders)

→ Clean Architecture Backend Skill:
  1. Analyzes domain concepts (Order, cancel)
  2. Determines layer placement (Domain Layer)
  3. Assigns responsibilities (Order.cancel() method)
  4. Implements with proper dependency direction
  5. Generates complete code for all layers
```

## Project Structure

```text
.claude/
├── agents/
│   ├── ddd-architecture-orchestrator.md    # Strategic orchestrator
│   ├── backend-architecture-reviewer.md    # Architecture review
│   ├── be-architect.md                     # Backend implementation
│   └── requirements-analyst.md             # Requirements analysis
│
├── skills/
│   ├── requirements-analysis/              # ✅ DDD requirements analysis
│   │   ├── SKILL.md
│   │   ├── references/
│   │   │   ├── DDD_PATTERNS.md
│   │   │   ├── UBIQUITOUS_LANGUAGE_GUIDE.md
│   │   │   ├── ANALYSIS_TEMPLATE.md
│   │   │   └── DOMAIN_DESIGN_GUIDE.md
│   │   └── assets/
│   │       └── requirements-analysis-template.md
│   │
│   ├── clean-architecture-backend/         # ✅ Automated architecture decisions
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── ARCHITECTURE_DECISION_FRAMEWORK.md
│   │       ├── RESPONSIBILITY_DRIVEN_DESIGN.md
│   │       ├── DEPENDENCY_MANAGEMENT.md
│   │       └── CODE_REVIEW_CHECKLIST.md
│   │
│   ├── commit-message/                     # ✅ Git commit message generation
│   │
│   ├── architecture-design/                # 🚧 Architecture design patterns (Work in Progress)
│   ├── spec-to-tasks/                      # 🚧 Specification to task breakdown (Work in Progress)
│   └── writing-backend-code/               # 🚧 Backend code writing patterns (Work in Progress)
│
└── settings.local.json                     # Local configuration
```

### Skill Status

**✅ Production Ready Skills:**

1. **requirements-analysis**: Fully functional DDD-based requirements analysis with comprehensive reference materials including DDD patterns, ubiquitous language guide, and analysis templates.

2. **clean-architecture-backend**: Automated architectural decision-making with layer placement, dependency management, responsibility assignment following GRASP principles, and comprehensive code review checklist.

3. **commit-message**: Git commit message generation following Korean language conventions with semantic commit types and structured format.

**🚧 Work in Progress Skills:**

The following skills are currently under development and planned for future releases:

- **architecture-design**: Architecture design patterns and documentation tools
- **spec-to-tasks**: Specification to development task breakdown automation
- **writing-backend-code**: Backend code writing patterns and implementation guidelines

These incomplete skills will provide additional capabilities for the development workflow once completed.

---

**Built with principles, designed for maintainability, implemented with precision.**
