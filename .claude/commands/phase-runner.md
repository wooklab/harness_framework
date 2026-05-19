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
2. **자기완결성** — 각 step 파일은 독립된 sub-agent 세션에서 실행된다. "이전 대화에서 논의한 바와 같이" 같은 외부 참조는 금지한다. 필요한 정보는 전부 파일 안에 적는다.
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
    {
      "step": 0,
      "name": "project-setup",
      "status": "pending",
    },
    {
      "step": 1,
      "name": "domain-model",
      "status": "pending"
    },
    {
      "step": 2,
      "name": "api-layer",
      "status": "pending"
    }
  ]
}
```

완료된 step 예시:

```json
{
  "step": 0,
  "name": "project-setup",
  "status": "completed",
  "started_at": "2026-05-13T12:16:32+0900",
  "summary": "산출물 한 줄 요약",
  "completed_at": "2026-05-13T12:45:10+0900"
}
```

필드 규칙:

- `project`: 프로젝트명 (CLAUDE.md 참조).
- `phase`: Jira 티켓 번호 (예: `SELL-2610`). 디렉토리명과 일치시킨다.
- `ticket`: 브랜치명 생성용. 보통 `phase`와 동일.
- `steps[].step`: 0부터 시작하는 순번.
- `steps[].name`: kebab-case slug.
- `steps[].status`: 초기값은 모두 `"pending"`.

상태 전이와 자동 기록 필드:

| 전이 | 기록되는 필드 | 기록 주체 |
|------|-------------|----------|
| → `completed` | `summary` | step-executor (sub-agent) |
| → `completed` | `completed_at` | phase-runner (메인) |
| → `error` | `error_message` | step-executor |
| → `error` | `failed_at` | phase-runner |
| → `blocked` | `blocked_reason` | step-executor |
| → `blocked` | `blocked_at` | phase-runner |

`summary`는 step 완료 시 산출물을 한 줄로 요약한 것으로, 메인이 다음 step 프롬프트에 컨텍스트로 누적 전달한다. 따라서 다음 step에 유용한 정보(생성된 파일, 핵심 결정 등)를 담아야 한다.

`created_at`은 메인이 최초 실행 시 task 레벨에 한 번만 기록한다. step 레벨의 `started_at`도 메인이 각 step 시작 시 자동 기록한다. 생성 시 넣지 않는다.

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

### E. 실행 (phase-runner 오케스트레이션)

메인 세션이 직접 오케스트레이션한다. 각 step의 구현은 **`step-executor` 서브에이전트를 Agent 도구로 spawn**하여 격리된 컨텍스트에서 실행한다. `claude -p`나 외부 스크립트는 사용하지 않는다.

사용자가 "phase-runner 실행" / "{ticket} 실행" / 유사 의도를 표명하면 아래 절차를 수행한다. `--push` 옵션 의사가 있으면 마지막에 push까지 진행.

#### E-1. 실행 전 준비

1. **phase 디렉토리 확인**: `phases/{ticket-no}/index.json` 존재 여부 확인. 없으면 사용자에게 보고하고 중단.

2. **메타데이터 로드**: `index.json`에서 `project`, `phase`, `ticket`, `steps` 추출. `ticket`이 없으면 `phase` 값을 사용.

3. **차단 상태 확인**: `steps`를 역순으로 훑어 첫 비-pending step의 상태를 본다.
   - `error` → 사용자에게 `error_message` 보고 후 중단. "status를 pending으로 되돌리고 error_message를 지운 뒤 재실행하세요" 안내.
   - `blocked` → `blocked_reason` 보고 후 중단. "사유 해결 후 status를 pending으로 되돌리고 blocked_reason을 지운 뒤 재실행하세요" 안내.
   - 그 외(`completed` 또는 없음) → 정상 진행.

4. **브랜치 정리**:
   ```bash
   git rev-parse --abbrev-ref HEAD
   ```
   현재 브랜치가 `feature/{ticket}`이 아니면 checkout한다. 존재하면 `git checkout feature/{ticket}`, 없으면 `git checkout -b feature/{ticket}`.

5. **created_at 기록**: `index.json`에 `created_at` 키가 없으면 현재 KST(`+09:00`) 타임스탬프를 `YYYY-MM-DDTHH:MM:SS+0900` 형식으로 기록.

6. **가드레일 수집**: 다음을 메모리에 로드한다 (각 step spawn 시 프롬프트에 inline 주입).
   - `CLAUDE.md` 전문
   - `docs/*.md` 전체 (있는 만큼)

#### E-2. Step 실행 루프

`steps` 배열의 첫 `pending` step부터 순서대로 다음을 수행. 모두 `completed`가 될 때까지 반복.

각 step마다:

##### (a) started_at 기록
해당 step에 `started_at`이 없으면 현재 KST 타임스탬프 기록 후 저장.

##### (b) 컨텍스트 조립
- 이전 완료 step들의 `summary` 목록 추출 (status="completed" && summary 있는 항목)
- 직전 시도가 있었다면 그 시도의 `error_message`

##### (c) step-executor 서브에이전트 spawn

Agent 도구로 `subagent_type: step-executor` 호출. 프롬프트 구조:

```
당신은 {project} 프로젝트의 step-executor 에이전트입니다.

티켓: {ticket}
대상 step 파일: phases/{ticket}/step{N}.md
step 번호: {N}
step 이름: {name}

## 가드레일

### CLAUDE.md

<CLAUDE.md 전문>

### docs/{파일명}

<docs/*.md 전문 — 파일별로 섹션 분리>

---

## 이전 Step 산출물 (있을 경우만)

- Step 0 (project-setup): <summary>
- Step 1 (domain-model): <summary>
...

---

## ⚠ 이전 시도 실패 — 반드시 참고하여 수정 (재시도일 때만)

<직전 error_message 전문>

---

## 작업 지시

phases/{ticket}/step{N}.md 를 끝까지 수행하고 종료 보고 포맷대로 반환하세요.
git 명령은 절대 실행하지 마세요 (메인 세션이 처리합니다).
```

##### (d) 결과 검증 (메인 세션이 직접)

서브에이전트 반환 후:

1. **index.json 재독**: 해당 step의 status를 확인.
2. **status별 처리**:

   **`completed`인 경우 — AC 재실행 검증**:
   - `phases/{ticket}/step{N}.md`에서 AC 커맨드를 파싱(```bash 블록의 첫 줄들).
   - 메인 세션이 Bash로 직접 AC를 다시 실행.
   - 종료 코드 0이면 진짜 completed로 인정. (e)로 진행.
   - 0이 아니면 거짓 완료 처리: status를 `pending`으로 되돌리고, error_message에 "거짓 완료: AC 재검증 실패. 출력: \<출력 일부\>"를 기록. 재시도 카운트 증가.

   **`blocked`인 경우**:
   - `blocked_at` 타임스탬프 기록.
   - `phases/index.json`이 있으면 해당 phase status를 `blocked`로 업데이트.
   - 사용자에게 `blocked_reason` 보고 후 중단.

   **`error`인 경우 또는 status 변경 없음**:
   - 재시도 카운트 증가. error_message 보존.
   - (e)로 진행.

##### (e) 재시도 / 실패 확정

- 같은 step 시도 횟수 < 3:
  - status를 `pending`으로 되돌리고 error_message는 보존(다음 spawn 프롬프트에 주입).
  - 사용자에게 "↻ Step N retry M/3 — \<error 요약\>" 보고.
  - (b)부터 다시 시작.
- 시도 횟수 ≥ 3:
  - status를 `error`로 고정, `error_message`에 "[3회 시도 후 실패] \<마지막 에러\>" 기록.
  - `failed_at` 타임스탬프 기록.
  - 해당 step 커밋(아래 (f)) 수행 후 `phases/index.json`을 `error`로 업데이트하고 중단.

##### (f) 성공 시 커밋

`completed` 확정 시:

1. `completed_at` 타임스탬프 기록.
2. 코드 변경 커밋 (output 파일·index.json 제외):
   ```bash
   git add -A
   git reset HEAD -- phases/{ticket}/index.json
   # output.json은 이번 설계에서 사용하지 않으면 생략
   git diff --cached --quiet || git commit -m "feat({ticket}): step {N} {name}"
   ```
3. 메타데이터 커밋:
   ```bash
   git add -A
   git diff --cached --quiet || git commit -m "docs({ticket}): step {N} output"
   ```

#### E-3. 전체 완료 후

1. 모든 step이 `completed`면 `index.json`에 `completed_at` 기록.
2. `phases/index.json`(top-level)이 있으면 해당 phase status를 `completed`로, `completed_at`을 기록.
3. 잔여 메타데이터를 마무리 커밋:
   ```bash
   git add -A
   git diff --cached --quiet || git commit -m "docs({ticket}): phase {phase} 완료"
   ```
4. 사용자가 `--push` 또는 push 요청한 경우:
   ```bash
   git push -u origin feature/{ticket}
   ```
5. 최종 보고: 완료된 step 수, 소요 시간 요약, 브랜치명, push 여부.

#### E-4. 진행 보고 가이드

메인 세션은 각 step 진입·완료·재시도 시 한 줄로 사용자에게 보고:

- 시작: `▶ Step {N}/{total-1} ({done} done): {name}`
- 재시도: `↻ Step {N}: retry {M}/3 — {error 한 줄}`
- 완료: `✓ Step {N}: {name}`
- 차단: `⏸ Step {N}: {name} blocked — {reason}`
- 실패: `✗ Step {N}: {name} failed after 3 attempts — {error}`

#### E-5. 에러 복구 (사용자 가이드)

- **error 발생 시**: `phases/{ticket-no}/index.json`에서 해당 step의 `status`를 `"pending"`으로 바꾸고 `error_message`, `failed_at`을 삭제한 뒤 phase-runner를 재호출한다.
- **blocked 발생 시**: `blocked_reason`에 적힌 사유를 해결한 뒤, `status`를 `"pending"`으로 바꾸고 `blocked_reason`, `blocked_at`을 삭제한 뒤 재호출한다.

---

## 부록: 절대 위반 금지

- 메인 세션은 절대 step 코드를 직접 작성하지 마라. 반드시 `step-executor` 서브에이전트를 spawn해 위임한다. 컨텍스트 격리가 이 스킬의 핵심이다.
- 서브에이전트가 "completed"라고 보고해도, AC를 메인 세션이 직접 재실행해 통과를 확인하기 전까지는 신뢰하지 마라.
- step-executor가 "읽은 파일" 목록에 step의 "읽어야 할 파일"을 빠짐없이 포함했는지 확인하라. 누락 시 재시도 사유로 사용.
- `claude -p` 또는 `scripts/execute.py`는 호출하지 마라. 이 스킬이 직접 오케스트레이션한다.
