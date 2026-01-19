# [기능명] 개발 태스크

> 생성일: YYYY-MM-DD
> 요구사항 문서: [링크 또는 참조]

---

## 요약

| 항목 | 내용 |
|------|------|
| 기능 | [기능 한 줄 설명] |
| 총 태스크 수 | Domain: X, Application: X, Infrastructure: X, Presentation: X |
| 예상 구현 순서 | Domain → Application → Infrastructure → Presentation |

---

## 입력 정보

### 확정된 요구사항

- [요구사항 1]
- [요구사항 2]

### 정책/규칙

| 정책 ID | 정책명 | 규칙 |
|---------|--------|------|
| P-01 | [정책명] | [규칙 설명] |
| P-02 | [정책명] | [규칙 설명] |

### 흐름 (Use Case 시나리오)

```
1. [Step 1]
   ↓
2. [Step 2]
   ↓
3. [Step 3]
```

### 상태 전이

```
[Initial State] → [State 1] → [State 2] → [Final State]
```

---

## Domain Layer Tasks

### Entity/Aggregate

- [ ] **[Entity명]** 엔티티 생성
  - 식별자: `[ID 타입]`
  - 속성:
    - `[속성1]`: [타입] - [설명]
    - `[속성2]`: [타입] - [설명]
  - 불변식:
    - [Invariant 1]
    - [Invariant 2]
  - 생성 규칙: [팩토리 메서드 또는 생성 조건]

### Value Object

- [ ] **[VO명]** 값 객체 생성
  - 속성:
    - `[속성1]`: [타입]
    - `[속성2]`: [타입]
  - 검증 규칙:
    - [Validation 1]
    - [Validation 2]
  - 동등성: [비교 기준]

### Domain Policy

- [ ] **[Policy명]** 정책 구현
  - 입력: `[Input 타입]`
  - 출력: `[Output 타입]`
  - 규칙:
    ```
    IF [조건]
    THEN [결과]
    ELSE [대안]
    ```
  - 참조 정책 ID: P-XX

### Domain Event

- [ ] **[Event명]** 이벤트 정의
  - 발행 조건: [Trigger 조건]
  - 페이로드:
    - `[필드1]`: [타입]
    - `[필드2]`: [타입]
  - 구독자: [예상 구독자 목록]

### Domain Service

- [ ] **[Service명]** 도메인 서비스 구현
  - 책임: [여러 Aggregate 조합 로직]
  - 입력: `[Input 타입]`
  - 출력: `[Output 타입]`
  - 조합 대상: [Aggregate 목록]

---

## Application Layer Tasks

### Use Case

- [ ] **[UseCase명]** Use Case 구현
  - Command:
    ```
    [CommandName]
    - [필드1]: [타입]
    - [필드2]: [타입]
    ```
  - Result:
    ```
    [ResultName]
    - [필드1]: [타입]
    - [필드2]: [타입]
    ```
  - 흐름:
    1. [Step 1 - 검증/조회]
    2. [Step 2 - 도메인 로직 실행]
    3. [Step 3 - 저장/이벤트 발행]
  - 트랜잭션 범위: [범위 설명]
  - 의존 Port: [Port 목록]

### Port (Outbound)

- [ ] **[Port명]** Port 인터페이스 정의
  - 메서드:
    ```
    [반환타입] [메서드명]([파라미터])
    ```
  - 목적: [외부 의존성 추상화 대상]
  - 구현체: [예상 Adapter 위치]

---

## Infrastructure Layer Tasks

### Repository

- [ ] **[Repository명]** Repository 구현
  - 구현 대상 Port: `[Port명]`
  - 저장소: [DB/Cache 종류]
  - 쿼리:
    | 메서드 | 쿼리 설명 | 인덱스 필요 |
    |--------|-----------|-------------|
    | `findById` | [설명] | [Y/N] |
    | `save` | [설명] | - |
  - 매핑: Entity ↔ JPA Entity (또는 Document)

### Adapter (External API)

- [ ] **[Adapter명]** Adapter 구현
  - 구현 대상 Port: `[Port명]`
  - 외부 시스템: [연동 대상 시스템명]
  - API 스펙:
    - Endpoint: `[HTTP Method] [URL]`
    - Request: [요청 형식]
    - Response: [응답 형식]
  - 매핑: External DTO ↔ Domain
  - 에러 처리: [에러 매핑 전략]

### Event Publisher/Handler

- [ ] **[Publisher명]** 이벤트 발행자 구현
  - 발행 이벤트: `[Event명]`
  - 발행 방식: [동기/비동기, 메시지 브로커]
  - 재시도 정책: [재시도 전략]

- [ ] **[Handler명]** 이벤트 핸들러 구현
  - 처리 이벤트: `[Event명]`
  - 동작: [처리 내용]
  - 멱등성: [멱등성 보장 방법]

---

## Presentation Layer Tasks

### Controller

- [ ] **[Controller명]** Controller 구현
  - Endpoint: `[HTTP Method] [Path]`
  - Request DTO: `[RequestDTO명]`
  - Response DTO: `[ResponseDTO명]`
  - 호출 UseCase: `[UseCase명]`
  - 인증/인가: [필요 권한]
  - 에러 응답:
    | 상황 | HTTP Status | Error Code |
    |------|-------------|------------|
    | [상황1] | [4XX/5XX] | [CODE] |

### DTO

- [ ] **[RequestDTO명]** Request DTO 정의
  - 필드:
    - `[필드1]`: [타입] - [검증 규칙]
    - `[필드2]`: [타입] - [검증 규칙]

- [ ] **[ResponseDTO명]** Response DTO 정의
  - 필드:
    - `[필드1]`: [타입]
    - `[필드2]`: [타입]

---

## 구현 순서 체크리스트

### Phase 1: Domain Layer
- [ ] Value Object 구현
- [ ] Entity/Aggregate 구현
- [ ] Domain Policy 구현
- [ ] Domain Service 구현 (필요 시)
- [ ] Domain Event 정의

### Phase 2: Application Layer
- [ ] Port 인터페이스 정의
- [ ] Use Case 구현
- [ ] Unit 테스트 작성

### Phase 3: Infrastructure Layer
- [ ] Repository 구현
- [ ] External Adapter 구현 (필요 시)
- [ ] Event Handler 구현 (필요 시)
- [ ] Integration 테스트 작성

### Phase 4: Presentation Layer
- [ ] DTO 정의
- [ ] Controller 구현
- [ ] API 테스트 작성

---

## 참고 사항

### 의존성 다이어그램

```
[Controller] → [UseCase] → [Domain Entity]
                  ↓              ↑
              [Port] ←――――― [Adapter/Repository]
```

### 관련 문서

- 요구사항 분석: [링크]
- 아키텍처 설계: [링크]
- API 명세: [링크]
