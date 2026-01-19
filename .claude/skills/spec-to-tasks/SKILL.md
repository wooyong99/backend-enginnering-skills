---
name: spec-to-tasks
description: 확정된 요구사항, 정책, 흐름, 상태 정보를 입력받아 Clean Architecture 레이어별 개발 태스크로 정형화하는 스킬. 요구사항 분석 완료 후 구현 전 태스크 분해가 필요할 때 사용. Domain/Application/Infrastructure/Presentation 레이어별로 Entity, VO, Policy, UseCase, Port, Repository, Adapter, Controller 등의 구현 태스크를 체계적으로 도출한다.
---

# Spec to Tasks

정리된 요구사항을 개발 태스크로 정형화하는 스킬.

## Input Requirements

이 스킬을 사용하기 전 다음 정보가 확정되어 있어야 한다:

| 항목 | 설명 | 예시 |
|------|------|------|
| 확정된 요구사항 | 구현할 기능 명세 | "프리미엄 회원은 무제한 다운로드" |
| 정책/규칙 | 비즈니스 규칙, 제약조건 | "일일 다운로드는 자정에 초기화" |
| 흐름 | Use Case 시나리오 | "1. 회원 등급 확인 → 2. 잔여 횟수 확인 → 3. 다운로드 실행" |
| 상태 | 도메인 상태, 상태 전이 | "PENDING → APPROVED → COMPLETED" |

## Output Structure

태스크를 Clean Architecture 레이어 기준으로 분류한다.

### 1. Domain Layer Tasks

비즈니스 규칙 구현 태스크.

```
## Domain Tasks

### Entity/Aggregate
- [ ] [Entity명] 엔티티 생성
  - 속성: [속성 목록]
  - 불변식: [Invariant 규칙]

### Value Object
- [ ] [VO명] 값 객체 생성
  - 속성: [속성 목록]
  - 검증 규칙: [Validation 규칙]

### Domain Policy
- [ ] [Policy명] 정책 구현
  - 입력: [Input 타입]
  - 출력: [Output 타입]
  - 규칙: [비즈니스 규칙]

### Domain Event
- [ ] [Event명] 이벤트 정의
  - 발행 조건: [Trigger 조건]
  - 페이로드: [Event 데이터]

### Domain Service
- [ ] [Service명] 도메인 서비스 구현
  - 책임: [여러 Aggregate 조합 로직]
```

### 2. Application Layer Tasks

Use Case 흐름 제어 태스크.

```
## Application Tasks

### Use Case
- [ ] [UseCase명] Use Case 구현
  - Command: [입력 데이터]
  - Result: [출력 데이터]
  - 흐름:
    1. [Step 1]
    2. [Step 2]
    3. [Step 3]

### Port (Outbound)
- [ ] [Port명] Port 인터페이스 정의
  - 메서드: [메서드 시그니처]
  - 목적: [외부 의존성 추상화 대상]
```

### 3. Infrastructure Layer Tasks

외부 시스템 연동 태스크.

```
## Infrastructure Tasks

### Repository
- [ ] [Repository명] Repository 구현
  - 구현 대상 Port: [Port명]
  - 저장소: [DB/Cache 등]
  - 쿼리:
    - [메서드1]: [쿼리 설명]
    - [메서드2]: [쿼리 설명]

### Adapter (External API)
- [ ] [Adapter명] Adapter 구현
  - 구현 대상 Port: [Port명]
  - 외부 시스템: [연동 대상]
  - 매핑: [DTO ↔ Domain 변환]

### Event Publisher/Handler
- [ ] [Handler명] 이벤트 핸들러 구현
  - 처리 이벤트: [Event명]
  - 동작: [처리 내용]
```

### 4. Presentation Layer Tasks

API 엔드포인트 태스크.

```
## API Tasks

### Controller
- [ ] [Controller명] Controller 구현
  - Endpoint: [HTTP Method] [Path]
  - Request: [Request DTO]
  - Response: [Response DTO]
  - 호출 UseCase: [UseCase명]

### DTO
- [ ] [DTO명] Request/Response DTO 정의
  - 필드: [필드 목록]
  - 검증: [Validation 규칙]
```

## Workflow

```
1. 요구사항 분석
   ↓
2. 도메인 개념 추출 (Entity, VO, Policy, Event)
   ↓
3. Use Case 흐름 정의
   ↓
4. 레이어별 태스크 도출
   ↓
5. 의존성 순서대로 태스크 정렬
   ↓
6. 태스크 문서 생성
```

### Task Ordering Rule

의존성 방향에 따라 구현 순서를 결정한다:

1. **Domain Layer** (의존성 없음, 먼저 구현)
   - Value Object → Entity → Policy → Domain Service → Domain Event

2. **Application Layer** (Domain에만 의존)
   - Port 정의 → Use Case 구현

3. **Infrastructure Layer** (Port 구현)
   - Repository → Adapter → Event Handler

4. **Presentation Layer** (Use Case 호출)
   - DTO → Controller

## Output

생성되는 태스크 문서 형식: [assets/template.md](assets/template.md)
