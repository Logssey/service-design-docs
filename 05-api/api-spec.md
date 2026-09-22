# API 명세

## Re:Used — API 명세

| 항목 | 값 |
| --- | --- |
| 기준 버전 | 1.0 |
| Base URL | `/api/v1` |

---

### 0. 공통 규약

#### 0.1. DTO 네이밍

엔티티를 직접 노출하지 않고 DTO로 변환하여 통신한다.

| 구분 | 접미사 | 예시 |
| --- | --- | --- |
| 요청 | `Request` | `ListingCreateRequest` |
| 응답 | `Response` | `ListingDetailResponse` |
| 목록 항목 | `Response` | `ListingSummaryResponse` |
| 중첩 요약 | `Response` | `UserSummaryResponse` |
| 페이지 래퍼 | `CursorPageResponse<T>` | `CursorPageResponse<ListingSummaryResponse>` |

패키지는 도메인별로 분리한다. `com.reused.listing.dto.request.ListingCreateRequest`

#### 0.2. 공통 DTO

```java
// CursorPageResponse<T>
{
  "items": [],
  "nextCursor": "eyJpZCI6MTIzfQ==",
  "hasNext": true
}

// UserSummaryResponse — 작성자·상대방 등 기본 중첩 표현
{
  "userId": 5,
  "nickname": "판매왕",
  "profileImageUrl": "https://..."
}

// CommunityAuthorResponse — 커뮤니티 작성자 표현
{
  "userId": 5,              // 탈퇴 작성자는 null
  "nickname": "재사용러"   // 탈퇴 작성자는 "탈퇴한 사용자"
}

// SellerBriefResponse — 게시글 상세에서 신뢰 정보를 함께 노출
{
  "userId": 5,
  "nickname": "판매왕",
  "profileImageUrl": "https://...",
  "completedTradeCount": 23,
  "averageRating": 4.7
}

// ListingBriefResponse — 거래·채팅에서 게시글을 요약 표현
{
  "listingId": 101,
  "title": "아이패드 프로 11인치",
  "price": 650000,
  "thumbnailUrl": "https://..."
}

// ErrorResponse
{
  "code": "FORBIDDEN",
  "message": "해당 리소스에 접근할 권한이 없습니다."
}
```

#### 0.3. 인증

| 구분 | 방식 |
| --- | --- |
| Access Token | `Authorization: Bearer {token}` |
| Refresh Token | HttpOnly·Secure 쿠키 (`refresh_token`) |

인증 표기: `—`(불필요, 선택적 토큰을 허용하면 별도 표기) / `USER` / `ADMIN`

커뮤니티 공개 조회 API는 Bearer 토큰 없이 호출할 수 있다. 유효한 토큰이 있으면 `isMine` 계산과 차단 사용자 콘텐츠 필터에 사용한다. `Authorization` 헤더를 보냈지만 토큰이 잘못되었거나 만료된 경우에는 익명 요청으로 무시하지 않고 `401 UNAUTHENTICATED`로 응답한다.

#### 0.4. 페이지네이션

커서 기반 (ADR-013).

**`CursorPageRequest`** — 쿼리 파라미터로 바인딩

| 필드 | 타입 | 기본값 | 설명 |
| --- | --- | --- | --- |
| `cursor` | String | null | 이전 응답의 `nextCursor` |
| `size` | Integer | 20 | 최대 100 |

#### 0.5. 오류 코드

| HTTP | code | 상황 |
| --- | --- | --- |
| 400 | `INVALID_INPUT` | 형식·길이·허용값 위반 |
| 401 | `UNAUTHENTICATED` | 토큰 없음 또는 만료 |
| 403 | `FORBIDDEN` | 역할 또는 소유권 불일치 |
| 403 | `USER_SUSPENDED` | 이용정지 상태 |
| 404 | `NOT_FOUND` | 리소스 없음 |
| 409 | `CONFLICT` | 현재 상태와 충돌 |
| 429 | `RATE_LIMITED` | 호출 빈도 초과 |
| 500 | `INTERNAL_ERROR` | 내부 오류 |
| 502 | `EXTERNAL_SERVICE_ERROR` | 외부 연동 실패 |
| 503 | `SERVICE_UNAVAILABLE` | 기능이 비활성화된 상태 |

오류 응답에 스택 트레이스, 쿼리, 내부 식별자를 포함하지 않는다.

#### 0.6. 공통 규칙

- 시간은 ISO 8601 UTC (`2026-03-15T09:30:00Z`)
- 필드명은 camelCase
- 소유권·당사자 검증은 서버에서 수행한다 (`NFR-AUTH-010`)
- 차단 필터는 **목록 조회에만** 적용한다. 중고거래 게시글 목록과 커뮤니티 게시글·댓글 목록에서 차단한 작성자의 콘텐츠를 제외하며, 직접 접근은 제한하지 않는다
- 커뮤니티 응답에서 탈퇴 작성자는 `CommunityAuthorResponse.userId=null`, `nickname="탈퇴한 사용자"`로 익명화한다
- 커뮤니티 게시글 목록은 `excerpt`, 상세는 원문 `content`를 반환하며 두 응답 모두 `category`, `author`, `commentCount`, `viewCount`, `isMine`, `createdAt`, `updatedAt`을 포함한다
- 커뮤니티 제목·본문·댓글의 길이는 앞뒤 공백을 제거한 값으로 검증하고, 제거된 값을 저장한다
- 이미지 URL은 비공개 S3 버킷에 대한 서명된 URL이다

### 1. WebSocket 이벤트

채팅 서버(Socket.IO)와의 통신 규약이다. 메시지 저장은 API 서버가 담당하므로, WebSocket은 **실시간 전달 전용**이다.

#### 연결 엔드포인트

```
wss://reused.app/socket.io
```

#### 인증 흐름

연결 수립 후 별도 인증 메시지로 토큰을 전달한다. 토큰을 쿼리 파라미터로 전달하지 않는다.

```
클라이언트                     채팅 서버
    |  connect                    |
    |---------------------------->|
    |                             | 인증 대기 상태
    |  emit("authenticate",       |
    |    { token: "eyJ..." })     |
    |---------------------------->|
    |                             | 서명·만료 검증
    |  emit("authenticated")      |
    |<----------------------------|
    |                             |
    |  emit("subscribe",          |
    |    { chatRoomId: 12 })      |
    |---------------------------->|
    |                             | 참여자 검증
```

**규칙**

- 연결 후 5초 내 `authenticate`가 없으면 연결을 종료한다
- 인증 실패 시 `auth_error` 발신 후 연결을 종료한다
- 인증 전에는 `subscribe`를 포함한 어떤 이벤트도 처리하지 않는다

#### 클라이언트 → 서버

| 이벤트 | 페이로드 | 설명 |
| --- | --- | --- |
| `authenticate` | `{ "token": "eyJ..." }` | Access Token 전달 |
| `subscribe` | `{ "chatRoomId": 12 }` | 채팅방 구독. 참여자 검증 수행 |
| `unsubscribe` | `{ "chatRoomId": 12 }` | 구독 해제 |

메시지 전송은 WebSocket이 아닌 `POST /chat-rooms/{id}/messages`를 사용한다. 저장 성공을 보장하기 위함이다.

#### 서버 → 클라이언트

| 이벤트 | 페이로드 | 설명 |
| --- | --- | --- |
| `authenticated` | 없음 | 인증 성공 |
| `auth_error` | `{ "reason": "EXPIRED" }` | 인증 실패. 연결 종료 |
| `forbidden` | `{ "chatRoomId": 12 }` | 참여자가 아닌 방 구독 시도 |
| `message` | `MessageResponse` | 새 메시지 수신 |
| `read` | `{ "chatRoomId": 12, "lastReadMessageId": 987 }` | 상대방이 읽음 |
| `message_deleted` | `{ "chatRoomId": 12, "messageId": 987 }` | 메시지 삭제됨 |

#### 재연결

연결이 끊긴 동안의 메시지는 `GET /chat-rooms/{id}/messages`로 조회하여 복구한다. 전달 실패가 저장을 되돌리지 않는다.

클라이언트는 현재 접속 중인 Origin을 기준으로 WebSocket URL을 구성한다. 별도 호스트를 사용하지 않으므로 Origin 검증과 쿠키 전달이 단순하다.

관련: `FR-CHAT-002`, `NFR-AUTH-011`, `NFR-AVAIL-005`

---

### 2. 엔드포인트 요약표

| 도메인 | 개수 | 경로 |
| --- | --- | --- |
| 인증 | 4 | `/auth/*` |
| 사용자 | 6 | `/users/*` |
| 카테고리 | 1 | `/categories` |
| 게시글 | 5 | `/listings/*` |
| 이미지 | 3 | `/images/*` |
| 관심 상품 | 3 | `/wishes`, `/listings/{id}/wish` |
| 거래 | 7 | `/trades/*` |
| 판매 관리 | 2 | `/me/*` |
| 채팅 | 6 | `/chat-rooms/*` |
| 커뮤니티 | 8 | `/community/posts/*` |
| 후기 | 1 | `/reviews` |
| 신고·차단 | 5 | `/reports/*`, `/blocks/*` |
| 알림 | 6 | `/notifications/*` |
| 챗봇 | 2 | `/chatbot/*` |
| 공지사항 | 2 | `/notices/*` |
| 관리자 | 14 | `/admin/*` |
| 합계 | **75** |  |

[엔드포인트 (DB) c80f026c8600432ab3aee1fed9235da2](catalog/endpoints.csv)
