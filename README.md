# Harness Framework

Claude Code를 활용해 Java Spring Boot 프로젝트를 단계적으로 자동 개발하는 오케스트레이션 프레임워크.

## 동작 방식

1. 작업을 여러 step으로 쪼개 `phases/` 디렉토리에 정의한다
2. `execute.py`가 각 step을 Claude Code에 순차 전달한다
3. Claude가 코드를 작성하고 `./gradlew build`로 검증한다
4. 검증 실패 시 최대 3회 자동 재시도한다
5. 성공하면 자동 커밋 후 다음 step으로 넘어간다

## 사전 요구사항

- [Claude Code](https://claude.ai/code) CLI 설치
- Python 3
- Java Spring Boot 프로젝트 (Gradle Wrapper 포함)

## 설치

프레임워크 파일을 Spring Boot 프로젝트 루트에 복사한다.

```bash
git clone https://github.com/wooklab/harness_framework.git
cp -r harness_framework/.claude        my-project/
cp -r harness_framework/docs           my-project/
cp -r harness_framework/scripts        my-project/
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

> `docs/PRD.md`는 프로젝트 목표와 핵심 기능을 기술한다. Claude가 매 step마다 컨텍스트로 읽는다.

## 사용법

### Step 1. Phase 설계 — `/harness`

Claude Code에서 `/harness`를 입력하면 Claude가 `docs/`를 읽고 구현 계획을 제안한다.
승인하면 아래 파일들을 자동 생성한다.

```
phases/
└── PROJ-1234/
    ├── index.json    # step 목록 및 상태
    ├── step0.md      # step 정의
    └── step1.md
```

### Step 2. 실행 — `execute.py`

```bash
python3 scripts/execute.py PROJ-1234          # 순차 실행
python3 scripts/execute.py PROJ-1234 --push   # 실행 후 원격 브랜치 push
```

실행 시 자동으로 처리되는 것:

- `feat-PROJ-1234` 브랜치 생성 및 checkout
- `CLAUDE.md` + `docs/*.md` 를 매 step 프롬프트에 가드레일로 주입
- 완료된 step의 산출물 요약을 다음 step 컨텍스트로 누적 전달
- 실패 시 최대 3회 자동 재시도
- step 완료마다 자동 커밋

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
│   └── commands/
│       ├── harness.md         # /harness 슬래시 커맨드
│       └── review.md          # /review 슬래시 커맨드
├── docs/
│   ├── PRD.md                 # 프로젝트 목표·기능 정의
│   ├── ADR.md                 # 기술 결정 기록
│   ├── ARCHITECTURE.md        # 디렉토리 구조·패턴·데이터 흐름
│   └── API_GUIDE.md           # REST API 설계 규약
├── scripts/
│   ├── execute.py             # 오케스트레이터
│   └── test_execute.py        # 단위 테스트
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

step 실패 시 `phases/PROJ-1234/index.json`에서 해당 step의 `status`를 `"pending"`으로 변경하고 재실행한다.

```json
{ "step": 1, "name": "domain-model", "status": "pending" }
```

```bash
python3 scripts/execute.py PROJ-1234
```
