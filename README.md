# Harness Framework

Claude Code를 활용해 Java Spring Boot 프로젝트를 단계적으로 자동 개발하는 오케스트레이션 프레임워크.

## 동작 방식

1. 작업을 여러 step으로 쪼개 `phases/` 디렉토리에 정의한다
2. `/phase-runner` 스킬이 각 step을 순차 실행한다 — 메인 세션이 오케스트레이션하고, 각 step의 구현은 `step-executor` 서브에이전트가 격리된 컨텍스트에서 수행한다
3. 서브에이전트가 코드를 작성하고 `./gradlew build`로 검증한다
4. 검증 실패 시 메인 세션이 AC를 재실행해 거짓 완료를 차단하며, 최대 3회까지 자동 재시도한다
5. 성공하면 메인 세션이 자동 커밋 후 다음 step으로 넘어간다

> 과거 `scripts/execute.py`는 `claude -p`를 사용해 동일한 흐름을 구현했으나, `claude -p` 사용량이 구독에서 분리되면서 **deprecated** 되었다. 스킬 기반 흐름을 사용하라.

## 사전 요구사항

- [Claude Code](https://claude.ai/code) CLI
- Java Spring Boot 프로젝트 (Gradle Wrapper 포함)

## 설치

프레임워크 파일을 Spring Boot 프로젝트 루트에 복사한다.

```bash
git clone https://github.com/wooklab/harness_framework.git
cp -r harness_framework/.claude        my-project/
cp -r harness_framework/docs           my-project/
cp    harness_framework/CLAUDE.md      my-project/
```

## 설정

복사 후 아래 세 파일의 placeholder를 채운다.

### 1. `CLAUDE.md` — 기술 스택 및 규칙

```markdown
## 기술 스택
- Spring Boot 3.3 (Java 21)
- Gradle (Kotlin DSL)
- Spring Data JPA + Hibernate
...

## 아키텍처 규칙
- CRITICAL: 모든 비즈니스 로직은 @Service 레이어에서만 처리
- CRITICAL: Entity는 외부로 노출하지 말 것. 응답은 항상 DTO로 변환
```

### 2. `docs/ADR.md` — 기술 결정 기록

ADR-001(언어·버전), ADR-002(아키텍처 패턴), ADR-003(영속성), ADR-004(테스트 스택)을 결정한다.

### 3. `docs/ARCHITECTURE.md` — 디렉토리 구조

ADR-002에서 선택한 패턴(Layered / Hexagonal / DDD) 기준으로 옵션 A/B/C 중 하나만 남기고 나머지를 삭제한다.

> `docs/PRD.md`는 프로젝트 목표와 핵심 기능을 기술한다. step-executor가 매 step마다 컨텍스트로 읽는다.

## 사용법

### Step 1. Phase 설계 — `/phase-runner`

Claude Code에서 `/phase-runner`를 입력하면 Claude가 `docs/`를 읽고 구현 계획을 제안한다.
승인하면 아래 파일들을 자동 생성한다.

```
phases/
└── PROJ-1234/
    ├── index.json    # step 목록 및 상태
    ├── step0.md      # step 정의
    └── step1.md
```

### Step 2. 실행 — `/phase-runner {ticket}`

설계가 끝났으면 같은 `/phase-runner` 스킬에 티켓 번호를 주어 실행을 지시한다.

```
/phase-runner PROJ-1234           # 순차 실행
/phase-runner PROJ-1234 --push    # 실행 후 원격 브랜치 push
```

실행 시 자동으로 처리되는 것:

- `feature/PROJ-1234` 브랜치 생성 및 checkout (메인 세션)
- `CLAUDE.md` + `docs/*.md` 를 매 step 서브에이전트 프롬프트에 가드레일로 주입
- 완료된 step의 산출물 요약을 다음 step 컨텍스트로 누적 전달
- 각 step은 `step-executor` 서브에이전트가 격리된 컨텍스트에서 구현
- 메인 세션이 AC 커맨드를 재실행해 거짓 완료 차단
- 실패 시 이전 에러를 다음 시도에 주입하며 최대 3회 자동 재시도
- step 완료마다 메인 세션이 자동 커밋 (코드 `feat` + 메타데이터 `docs` 분리)

### Step 3. 리뷰 — `/review`

```
/review
```

아키텍처 준수, 트랜잭션 경계, N+1 쿼리, DTO 분리, 예외 처리 등 9개 항목을 체크한다.

## 디렉토리 구조

```
my-project/
├── .claude/
│   ├── settings.json          # hooks 설정
│   ├── commands/
│   │   ├── phase-runner.md    # /phase-runner 슬래시 커맨드 (오케스트레이터)
│   │   └── review.md          # /review 슬래시 커맨드
│   └── agents/
│       └── step-executor.md   # step 실행 서브에이전트 정의
├── docs/
│   ├── PRD.md                 # 프로젝트 목표·기능 정의
│   ├── ADR.md                 # 기술 결정 기록
│   ├── ARCHITECTURE.md        # 디렉토리 구조·패턴·데이터 흐름
│   └── API_GUIDE.md           # REST API 설계 규약
├── scripts/
│   ├── execute.py             # [DEPRECATED] 과거 오케스트레이터
│   └── test_execute.py
├── phases/
│   └── PROJ-1234/
│       ├── index.json
│       └── step0.md
├── src/                       # Spring Boot 소스
├── build.gradle.kts
└── CLAUDE.md                  # 기술 스택·아키텍처 규칙
```

## Hooks

`.claude/settings.json`에 두 가지 hook이 설정되어 있다.

| Hook | 시점 | 동작 |
|------|------|------|
| `Stop` | Claude 응답 완료 후 | `./gradlew build` 실행. 빌드 실패 시 에러를 Claude에 주입해 자동 수정 유도 |
| `PreToolUse` | Bash 명령 실행 전 | `rm -rf`, `git push --force` 등 위험 명령 차단 |

## 에러 복구

step 실패 시 `phases/PROJ-1234/index.json`에서 해당 step의 `status`를 `"pending"`으로 변경하고 `error_message`(또는 `blocked_reason`)를 지운 뒤 `/phase-runner PROJ-1234`를 재호출한다.

```json
{ "step": 1, "name": "domain-model", "status": "pending" }
```
