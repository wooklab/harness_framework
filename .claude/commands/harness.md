이 프로젝트는 Harness 프레임워크를 사용한다. 아래 워크플로우에 따라 작업을 진행하라.

---

## 워크플로우

### A. 탐색

`/docs/` 하위 문서(PRD, ARCHITECTURE, ADR 등)를 읽고 프로젝트의 기획·아키텍처·설계 의도를 파악한다. 필요시 Explore 에이전트를 병렬로 사용한다.

### B. 논의

구현을 위해 구체화하거나 기술적으로 결정해야 할 사항이 있으면 사용자에게 제시하고 논의한다.

### C. Step 설계

사용자가 구현 계획 작성을 지시하면 여러 step으로 나뉜 초안을 작성해 피드백을 요청한다.

설계 원칙:

1. **Scope 최소화** — 하나의 step에서 하나의 레이어 또는 모듈만 다룬다. 여러 모듈을 동시에 수정해야 하면 step을 쪼갠다.
2. **자기완결성** — 각 step 파일은 독립된 서브에이전트 세션에서 실행된다. "이전 대화에서 논의한 바와 같이" 같은 외부 참조는 금지한다. 필요한 정보는 전부 파일 안에 적는다.
3. **사전 준비 강제** — 관련 문서 경로와 이전 step에서 생성/수정된 파일 경로를 명시한다. 세션이 코드를 읽고 맥락을 파악한 뒤 작업하도록 유도한다.
4. **시그니처 수준 지시** — 함수/클래스의 인터페이스만 제시하고 내부 구현은 에이전트 재량에 맡긴다. 단, 설계 의도에서 벗어나면 안 되는 핵심 규칙(멱등성, 보안, 데이터 무결성 등)은 반드시 명시한다.
5. **AC는 실행 가능한 커맨드** — "~가 동작해야 한다" 같은 추상적 서술이 아닌 `./gradlew build` 같은 실제 실행 가능한 검증 커맨드를 포함한다.
6. **주의사항은 구체적으로** — "조심해라" 대신 "X를 하지 마라. 이유: Y" 형식으로 적는다.
7. **네이밍** — step name은 kebab-case slug로, 해당 step의 핵심 모듈/작업을 한두 단어로 표현한다 (예: `project-setup`, `api-layer`, `auth-flow`).

### D. 파일 생성

사용자가 승인하면 아래 파일들을 생성한다.

#### D-1. `phases/{ticket-no}/index.json` (task 상세)

```json
{
  "project": "<프로젝트명>",
  "phase": "<ticket-no>",
  "ticket": "<ticket-no>",
  "steps": [
    { "step": 0, "name": "project-setup", "status": "pending" },
    { "step": 1, "name": "domain-model", "status": "pending" },
    { "step": 2, "name": "api-layer", "status": "pending" }
  ]
}
```

필드 규칙:

- `project`: 프로젝트명 (CLAUDE.md 참조).
- `phase`: Jira 티켓 번호 (예: `SELL-2610`). 디렉토리명과 일치시킨다.
- `ticket`: phase와 동일한 값.
- `steps[].step`: 0부터 시작하는 순번.
- `steps[].name`: kebab-case slug.
- `steps[].status`: 초기값은 모두 `"pending"`.

상태 전이와 자동 기록 필드:

| 전이 | 기록되는 필드 | 기록 주체 |
|------|-------------|----------|
| → `completed` | `completed_at`, `summary` | 서브에이전트 (summary), 오케스트레이터 (timestamp) |
| → `error` | `failed_at`, `error_message` | 서브에이전트 (message), 오케스트레이터 (timestamp) |
| → `blocked` | `blocked_at`, `blocked_reason` | 서브에이전트 (reason), 오케스트레이터 (즉시 중단) |

`summary`는 step 완료 시 산출물을 한 줄로 요약한 것으로, 오케스트레이터가 다음 step 서브에이전트 프롬프트에 누적 전달한다.

#### D-2. `phases/{ticket-no}/step{N}.md` (각 step마다 1개)

```markdown
# Step {N}: {이름}

## 읽어야 할 파일

먼저 아래 파일들을 읽고 프로젝트의 아키텍처와 설계 의도를 파악하라:

- `/docs/ARCHITECTURE.md`
- `/docs/ADR.md`
- {이전 step에서 생성/수정된 파일 경로}

이전 step에서 만들어진 코드를 꼼꼼히 읽고, 설계 의도를 이해한 뒤 작업하라.

## 작업

{구체적인 구현 지시. 파일 경로, 클래스/함수 시그니처, 로직 설명을 포함.
코드 스니펫은 인터페이스/시그니처 수준만 제시하고, 구현체는 에이전트에게 맡겨라.
단, 설계 의도에서 벗어나면 안 되는 핵심 규칙은 명확히 박아넣어라.}

## Acceptance Criteria

```bash
./gradlew build   # 컴파일 + 테스트 + 검증 일괄 통과
```

## 검증 절차

1. 위 AC 커맨드를 실행한다.
2. 아키텍처 체크리스트를 확인한다:
   - ARCHITECTURE.md 디렉토리 구조를 따르는가?
   - ADR 기술 스택을 벗어나지 않았는가?
   - CLAUDE.md CRITICAL 규칙을 위반하지 않았는가?
3. 결과에 따라 `phases/{ticket-no}/index.json`의 해당 step을 업데이트한다:
   - 성공 → `"status": "completed"`, `"summary": "산출물 한 줄 요약"`
   - 수정 3회 시도 후에도 실패 → `"status": "error"`, `"error_message": "구체적 에러 내용"`
   - 사용자 개입 필요 (API 키, 외부 인증, 수동 설정 등) → `"status": "blocked"`, `"blocked_reason": "구체적 사유"` 후 즉시 중단

## 금지사항

- {이 step에서 하지 말아야 할 것. "X를 하지 마라. 이유: Y" 형식}
- 기존 테스트를 깨뜨리지 마라
```

### E. 실행

사용자가 실행을 지시하면 아래 오케스트레이터 로직을 수행한다. 외부 스크립트 없이 이 세션이 직접 오케스트레이터 역할을 맡는다.

#### E-1. 사전 준비

```
1. feature/{ticket-no} 브랜치로 checkout (없으면 생성)
2. phases/{ticket-no}/index.json 읽기
3. error/blocked 상태 step이 있으면 중단하고 사용자에게 보고
4. created_at 없으면 현재 시각 기록
```

#### E-2. 가드레일 수집

매 step 서브에이전트에 주입할 컨텍스트를 준비한다:

```
- CLAUDE.md 전문
- docs/*.md 전문 (ARCHITECTURE, ADR, API_GUIDE 등)
```

#### E-3. Step 실행 루프

pending step이 없을 때까지 반복한다:

```
while (pending step 존재):
    1. index.json에서 첫 번째 pending step 선택
    2. started_at 기록
    3. 서브에이전트 스폰 (아래 E-4 참조)
    4. index.json 재읽기 → 해당 step의 status 확인
    5. completed → completed_at 기록 → 코드 커밋 → 다음 step
       blocked  → blocked_at 기록 → 사용자에게 보고 후 중단
       error/pending → 재시도 카운터 증가
                       3회 초과 시 error 확정 + failed_at 기록 → 커밋 → 중단
```

#### E-4. 서브에이전트 프롬프트 구성

Agent tool을 사용해 서브에이전트를 스폰한다. 각 서브에이전트는 격리된 컨텍스트에서 실행된다.

프롬프트는 아래 순서로 조립한다:

```
[역할 선언]
당신은 {project} 프로젝트의 개발자입니다. 아래 step을 수행하세요.

[가드레일 — CLAUDE.md + docs/*.md 전문]

---

[이전 step 산출물 — completed step의 summary 목록]
## 이전 Step 산출물
- Step 0 (project-setup): {summary}
- Step 1 (domain-model): {summary}

---

[실패 피드백 — 재시도 시에만 포함]
## ⚠ 이전 시도 실패 — 아래 에러를 반드시 참고하여 수정하라
{error_message}

---

[작업 규칙]
1. 이전 step에서 작성된 코드를 확인하고 일관성을 유지하라.
2. 이 step에 명시된 작업만 수행하라. 추가 기능이나 파일을 만들지 마라.
3. 기존 테스트를 깨뜨리지 마라.
4. AC(Acceptance Criteria) 검증을 직접 실행하라.
5. phases/{ticket-no}/index.json의 해당 step status를 업데이트하라:
   - AC 통과 → "completed" + "summary" 필드에 산출물 한 줄 요약
   - 3회 수정 시도 후에도 실패 → "error" + "error_message" 기록
   - 사용자 개입 필요 시 → "blocked" + "blocked_reason" 기록 후 즉시 중단
6. 모든 변경사항을 커밋하라: feat({ticket}): step {N} {name}

---

[step 파일 내용 — phases/{ticket-no}/step{N}.md 전문]
```

#### E-5. 커밋 규칙

오케스트레이터(이 세션)가 각 step 완료 후 수행한다:

```
git add -A
# 코드 변경분 커밋
git commit -m "feat({ticket}): step {N} {name}"

# index.json 등 메타데이터 커밋
git add phases/{ticket-no}/
git commit -m "docs({ticket}): step {N} output"
```

#### E-6. 완료 처리

모든 step이 completed 상태가 되면:

```
1. index.json에 completed_at 기록
2. 최종 커밋: docs({ticket}): phase {phase} 완료
3. --push 옵션이 있으면 origin/{branch} 으로 push
4. 완료 요약 출력
```

#### E-7. 에러 복구

- **error 발생 시**: `phases/{ticket-no}/index.json`에서 해당 step의 `status`를 `"pending"`으로 바꾸고 `error_message`를 삭제한 뒤 `/harness`를 다시 실행한다.
- **blocked 발생 시**: `blocked_reason`에 적힌 사유를 해결한 뒤, `status`를 `"pending"`으로 바꾸고 `blocked_reason`을 삭제한 뒤 `/harness`를 다시 실행한다.
