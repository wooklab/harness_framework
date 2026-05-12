# 아키텍처

> 아키텍처 패턴은 **ADR-002**에서 결정한다.
> 결정 후 아래 디렉토리 구조 / 패턴 / 데이터 흐름 섹션을 선택한 패턴 기준으로 채운다.
> 아래 옵션 A/B/C는 참고용 예시이며, 결정 후 선택한 하나만 남기고 나머지를 삭제한다.

## 디렉토리 구조

```
src/
├── main/
│   ├── java/<base-pkg>/
│   │   {아키텍처 패턴에 맞는 하위 패키지 구조 — ADR-002 결정 후 채움}
│   └── resources/
│       ├── application.yml
│       └── db/migration/   # 마이그레이션 SQL (Flyway/Liquibase — ADR-003 결정)
└── test/java/<base-pkg>/   # 프로덕션 코드와 동일한 패키지 구조 미러링
```

### 옵션 A — Layered Architecture (가장 흔함, 빠른 시작)

```
<base-pkg>/
├── controller/     # @RestController — 라우팅·직렬화·Bean Validation
├── service/        # @Service — 비즈니스 로직, @Transactional 경계
├── domain/         # @Entity, 도메인 모델
├── repository/     # Spring Data JPA 인터페이스
├── dto/            # 요청/응답 DTO (Entity 노출 금지)
├── config/         # @Configuration, SecurityConfig 등
├── exception/      # @ControllerAdvice, 커스텀 예외
└── client/         # 외부 API 클라이언트 (WebClient/RestClient 래퍼)
```

### 옵션 B — Hexagonal / Ports & Adapters (프레임워크 격리, 테스트 용이)

```
<base-pkg>/
├── domain/                       # 순수 도메인 (Spring 의존 없음)
│   ├── model/                    # Entity, Value Object, Aggregate Root
│   ├── port/
│   │   ├── in/                   # UseCase 인터페이스 (driving port)
│   │   └── out/                  # Repository/Client 인터페이스 (driven port)
│   └── service/                  # UseCase 구현 (도메인 서비스)
├── adapter/
│   ├── in/
│   │   └── web/                  # @RestController (driving adapter)
│   └── out/
│       ├── persistence/          # JPA Repository 구현 (driven adapter)
│       └── client/               # 외부 API 어댑터
└── config/
```

### 옵션 C — DDD / Domain-Driven Design (복잡 도메인, Bounded Context)

```
<base-pkg>/
├── {bounded-context}/            # Bounded Context 단위로 최상위 패키지 분리
│   ├── domain/                   # Aggregate, Entity, Value Object, Domain Service
│   ├── application/              # Application Service, Command/Query 핸들러
│   ├── infrastructure/           # Repository 구현, 외부 시스템 어댑터
│   └── interfaces/               # @RestController, 메시지 컨슈머
└── shared/                       # 공유 커널 (cross-context VO, 공통 이벤트 등)
```

---

## 패턴

{ADR-002에서 선택한 아키텍처 패턴 이름}

핵심 규칙 (패턴 결정 후 해당하는 항목만 남길 것):

- **Layered**: Controller → Service → Repository 단방향 의존. 역방향 참조 금지.
- **Hexagonal**: `domain/` 패키지는 `spring-*` 라이브러리에 의존하지 않는다. `@Component` 계열 어노테이션은 `adapter/`에서만.
- **DDD**: Aggregate Root를 통해서만 내부 Entity 접근. Bounded Context 간 직접 객체 참조 금지 — 이벤트 또는 ID로만 통신.
- **공통**: DTO ↔ 도메인 객체 변환은 mapper에서만 (MapStruct 권장). `@Entity`를 HTTP 응답으로 직접 노출하지 않는다.

---

## 데이터 흐름

{패턴 결정 후 아래 중 하나를 선택해 채울 것}

**옵션 A — Layered**
```
HTTP Request
  → @RestController (입력 검증 @Valid)
  → @Service (@Transactional, 비즈니스 로직)
  → @Repository (JPA/DB 접근)
  → DTO 매핑 → JSON 응답
```

**옵션 B — Hexagonal**
```
HTTP Request
  → in/web Adapter (@RestController)
  → in port (UseCase 인터페이스 호출)
  → Domain Service (@Transactional, 비즈니스 로직)
  → out port (Repository/Client 인터페이스 호출)
  → out/persistence Adapter (JPA 구현)
  → DTO 매핑 → JSON 응답
```

**옵션 C — DDD**
```
HTTP Request
  → interfaces/ (@RestController → Command/Query 생성)
  → application/ Service (Command 처리, @Transactional)
  → domain/ Aggregate (비즈니스 규칙 적용)
  → infrastructure/ Repository (영속화)
  → DTO 매핑 → JSON 응답
```

---

## 트랜잭션 & 캐싱

**트랜잭션 경계** (패턴에 따라 위치가 다름):
- Layered: `@Transactional`은 `@Service` 메서드 레벨
- Hexagonal: UseCase 구현 메서드 레벨 (`domain/service/`)
- DDD: Application Service 메서드 레벨 (`application/`)
- 조회 전용 메서드는 `@Transactional(readOnly = true)` 적용

**캐싱** (필요 시):
- `@Cacheable` + {Redis / Caffeine 등 — ADR 결정}
- 캐시 레이어 위치: 패턴별 service/application 레이어

**인증 상태**:
- {Stateless JWT / Spring Session — ADR 결정}
