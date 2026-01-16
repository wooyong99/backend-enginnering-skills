---
name: commit-message
description: Generate collaborative Git commit messages following Conventional Commits specification. Use when creating commit messages, reviewing staged changes, or helping users write clear commit descriptions. Enforces type/scope/body separation and prioritizes "why" over "what" for better team collaboration and code review.
---

# Collaborative Commit Message Generator

Generate commit messages that help reviewers understand changes at a glance.

## Language Rule

**All commit messages MUST be written in Korean (한글).**
- Subject line: Korean
- Body: Korean
- Only type, scope, and technical terms (BREAKING CHANGE, Co-authored-by) remain in English

## Core Principles

1. **Why over What**: 변경 사항보다 변경 이유를 먼저 설명
2. **Reviewer-first**: 코드를 보지 않은 사람을 위해 작성
3. **Atomic commits**: 하나의 커밋에 하나의 논리적 변경

## Commit Format

```
<type>(<scope>): <한글 제목>

<한글 본문>

<footer>
```

### Structure Rules

- **Subject line**: 최대 50자, 명령형, 마침표 없음
- **Body**: 72자에서 줄바꿈, 제목 후 빈 줄
- **Footer**: 이슈 참조, breaking changes, co-authors

## Workflow

1. `git diff --staged`로 변경 사항 분석
2. 주요 의도 파악 (왜 이 변경이 필요한가?)
3. 적절한 type과 scope 선택
4. 의도를 담은 제목 작성
5. 맥락과 이유를 설명하는 본문 추가
6. Footer에 참조 정보 포함

## Writing the Body

본문은 다음 질문에 답하도록 구성:

```
왜 이 변경이 필요했는가?
→ [이 변경을 유발한 문제 또는 맥락]

어떻게 문제를 해결하는가?
→ [접근 방식에 대한 간략한 설명]

어떤 부수 효과가 있는가?
→ [리뷰어가 알아야 할 영향]
```

## Type Selection

| Type | 사용 시점 |
|------|----------|
| `feat` | 사용자를 위한 새 기능 |
| `fix` | 사용자를 위한 버그 수정 |
| `docs` | 문서만 변경 |
| `style` | 포맷팅, 로직 변경 없음 |
| `refactor` | 코드 변경, 기능/버그 수정 아님 |
| `perf` | 성능 개선 |
| `test` | 테스트 추가/수정 |
| `chore` | 빌드, 도구, 의존성 |
| `ci` | CI/CD 설정 |

For detailed type guidance: [references/commit-types.md](references/commit-types.md)

## Scope Guidelines

Scope는 영향받는 영역을 나타냄:

- 모듈명: `feat(auth)`, `fix(payment)`
- 레이어: `refactor(api)`, `test(service)`
- 기능: `feat(user-profile)`, `fix(checkout)`

전역적이거나 불명확한 경우 scope 생략.

## Quick Examples

**Good - 이유를 설명:**
```
fix(auth): 공용 네트워크에서 세션 하이재킹 방지

JWT 토큰이 secure 플래그 없이 전송되어
암호화되지 않은 연결에서 가로채기가 가능했음.

모든 인증 쿠키에 secure 및 httpOnly 플래그 추가.

Fixes #234
```

**Bad - 무엇만 설명:**
```
fix(auth): 쿠키에 secure 플래그 추가
```

For more examples: [references/examples.md](references/examples.md)

## Breaking Changes

Footer에 breaking changes 표시:

```
feat(api): 사용자 엔드포인트 응답 형식 변경

BREAKING CHANGE: GET /users가 페이지네이션 응답을 반환함.
클라이언트는 `data` 배열과 `meta` 페이지네이션 객체를 처리해야 함.

마이그레이션 가이드: docs/migration/v2-users.md
```

## Co-authoring

페어 프로그래밍 또는 협업 커밋:

```
feat(search): 퍼지 매칭 알고리즘 구현

Co-authored-by: Name <email@example.com>
Co-authored-by: Claude <noreply@anthropic.com>
```
