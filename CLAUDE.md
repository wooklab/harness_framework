# 프로젝트: {프로젝트명}

## 기술 스택
- Spring Boot {버전} (Java {버전})
- Gradle (Kotlin DSL)
- {ORM/영속성 예: Spring Data JPA + Hibernate, 또는 MyBatis, jOOQ}
- {인증 예: Spring Security + JWT}
- {기타: Lombok, MapStruct, Flyway 등}

## 아키텍처 규칙
- CRITICAL: {절대 지켜야 할 규칙 1 (예: 모든 비즈니스 로직은 @Service 레이어에서만 처리. @RestController에는 라우팅/직렬화/검증만 둘 것.)}
- CRITICAL: {절대 지켜야 할 규칙 2 (예: DB 접근은 Repository 인터페이스만 사용. 컨트롤러나 서비스에서 EntityManager 직접 사용 금지.)}
- {일반 규칙 (예: Entity는 외부로 노출하지 말 것. 응답은 항상 DTO로 변환.)}

## 개발 프로세스
- CRITICAL: 새 기능 구현 시 반드시 테스트를 먼저 작성하고, 테스트가 통과하는 구현을 작성할 것 (TDD)
- 커밋 메시지는 conventional commits 형식을 따를 것 (feat:, fix:, docs:, refactor:)

## 명령어
./gradlew bootRun   # 개발 서버
./gradlew build     # 컴파일 + 테스트 + check 일괄 (배포 산출물 포함)
./gradlew check     # 테스트 + 코드 품질 검증 (배포 산출물 제외)
./gradlew test      # 테스트만
