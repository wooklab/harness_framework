# Architecture Decision Records

## 철학
{프로젝트의 핵심 가치관 (예: MVP 속도 최우선. 외부 의존성 최소화. 작동하는 최소 구현을 선택.)}

---

### ADR-001: 언어 · 프레임워크 · 빌드툴

**결정**: Java {버전} + Spring Boot {버전} + Gradle (Kotlin DSL)

**이유**: {왜 이 버전 조합을 선택했는지}

**트레이드오프**:
- Spring Boot 3.x 선택 시: Java 17+ 필수, `javax.*` → `jakarta.*` 패키지 전환 필요
- Spring Boot 2.x 선택 시: Java 8~17 지원, LTS 지원이 2025년 11월 종료 예정
- Gradle Kotlin DSL: IDE 자동완성·타입 안전성 이점, Groovy DSL보다 초기 학습비용 소폭 높음

---

### ADR-002: 아키텍처 패턴

**결정**: {Layered Architecture / Hexagonal (Ports & Adapters) / DDD / Clean Architecture / Modular Monolith 중 선택}

**이유**: {왜 이 패턴을 선택했는지}

**트레이드오프**:
- **Layered**: 빠른 시작·낮은 학습비용. 도메인이 커지면 레이어 간 의존 방향 관리 필요.
- **Hexagonal**: 프레임워크 비의존 도메인 → 테스트 용이. 초기 보일러플레이트 증가.
- **DDD**: 복잡한 도메인 경계를 Bounded Context로 명확히 분리. 설계 선행 비용 큼.
- **Modular Monolith**: 명확한 모듈 경계로 마이크로서비스 전환 준비. 초기 설계 비용 중간.

> 결정 후 `ARCHITECTURE.md`의 디렉토리 구조 / 패턴 / 데이터 흐름 섹션을 이 패턴 기준으로 채울 것.

---

### ADR-003: 영속성 · 마이그레이션

**결정**: {Spring Data JPA + Hibernate / MyBatis / jOOQ 등} + {Flyway / Liquibase}

**이유**: {왜 이 ORM과 마이그레이션 도구를 선택했는지}

**트레이드오프**:
- **JPA**: 객체-관계 매핑 자동화, 복잡 쿼리는 JPQL/QueryDSL 필요
- **MyBatis**: SQL 직접 제어, 동적 쿼리 유연함, 매핑 코드 증가
- **jOOQ**: 타입 안전 SQL DSL, 스키마 변경 시 컴파일 타임 감지
- **Flyway**: SQL 파일 기반, 간단하고 직관적
- **Liquibase**: XML/YAML/JSON 지원, 롤백 기능 내장

---

### ADR-004: 테스트 스택

**결정**: JUnit 5 + AssertJ + Mockito {+ Testcontainers 여부}

**이유**: {왜 이 스택을 선택했는지}

**트레이드오프**:
- **Testcontainers 사용 시**: 실제 DB/외부 서비스 기반 통합 테스트 → 정확도 높음, CI 실행 시간 증가
- **Testcontainers 미사용 시**: H2 인메모리 DB 사용 → 빠른 실행, DB 방언 차이 주의
