# API 설계 가이드

## URI 명명 규약

| 규칙 | 올바른 예 | 잘못된 예 |
|------|-----------|-----------|
| 리소스는 복수 명사 | `/users`, `/orders` | `/user`, `/getUser` |
| kebab-case 사용 | `/order-items` | `/orderItems`, `/order_items` |
| 동사 금지 (HTTP 메서드로 표현) | `DELETE /users/{id}` | `/deleteUser/{id}` |
| 계층 관계는 중첩 경로 | `/users/{id}/orders` | `/getUserOrders?userId=` |

## HTTP 메서드 · 상태 코드 매핑

| 메서드 | 용도 | 성공 응답 |
|--------|------|-----------|
| `GET` | 단건/목록 조회 | `200 OK` |
| `POST` | 리소스 생성 | `201 Created` + `Location` 헤더 |
| `PUT` | 리소스 전체 교체 | `200 OK` 또는 `204 No Content` |
| `PATCH` | 리소스 부분 수정 | `200 OK` 또는 `204 No Content` |
| `DELETE` | 리소스 삭제 | `204 No Content` |

## 에러 응답 포맷 — RFC 7807 Problem Details

Content-Type: `application/problem+json`

```json
{
  "type": "https://example.com/errors/not-found",
  "title": "Resource Not Found",
  "status": 404,
  "detail": "User with id 42 does not exist.",
  "instance": "/users/42"
}
```

- `type`: 에러 유형 URI (문서 링크 또는 고정 식별자)
- `title`: 에러 유형의 사람이 읽을 수 있는 이름 (고정 문자열)
- `status`: HTTP 상태 코드
- `detail`: 이 요청에 특화된 에러 설명
- `instance`: 에러가 발생한 요청 URI

## 페이지네이션

**Offset 방식** (기본, 소규모):

```
GET /users?page=0&size=20&sort=createdAt,desc
```

응답:
```json
{
  "content": [...],
  "page": { "number": 0, "size": 20, "totalElements": 100, "totalPages": 5 }
}
```

**Cursor 방식** (대규모, 실시간 피드):

```
GET /feed?cursor={opaque-token}&size=20
```

응답:
```json
{
  "items": [...],
  "nextCursor": "{opaque-token}",
  "hasNext": true
}
```

## 인증

```
Authorization: Bearer {JWT}
```

{인증 방식 상세 — ADR 결정 후 작성. 예: JWT 만료 시간, refresh token 전략 등}

## API 버전 정책

{URI prefix 방식 vs Accept 헤더 방식 중 선택}

- **URI prefix (권장)**: `/api/v1/users`, `/api/v2/users`
- **Accept 헤더**: `Accept: application/vnd.example.v2+json`

## OpenAPI / Swagger 문서화

- 라이브러리: `springdoc-openapi`
- 문서 경로: `/swagger-ui.html` (개발 환경만 노출)
- `@Operation`, `@ApiResponse`, `@Schema` 어노테이션으로 DTO 및 엔드포인트 설명 작성

## 입력 검증

- Bean Validation 어노테이션(`@NotNull`, `@Size`, `@Pattern` 등)은 **DTO에만** 적용
- 컨트롤러 파라미터에 `@Valid` 또는 `@Validated` 선언
- 검증 실패 시 `400 Bad Request` + RFC 7807 포맷 응답

## 시간 표현

- **모든 시간은 ISO 8601 형식** (`2024-01-15T09:30:00Z`)
- Java 타입: `Instant` (UTC 기준) 또는 `ZonedDateTime`
- 클라이언트 표시용 타임존 변환은 클라이언트 책임
