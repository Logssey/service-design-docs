# Redis 키 규약

> Redis를 공유 상태 저장소로 사용하는 결정은 ADR-010에 있다. 이 문서는 그 결정을 따르는 키 설계 규약이다.

## 규약이 필요한 이유

Redis에는 테이블도 스키마도 없다. 키 하나하나가 평평한 공간에 나열될 뿐이므로 **키 이름이 곧 스키마다.**

- API 서버와 채팅 서버가 같은 Redis를 사용한다. 규약이 없으면 키가 충돌하고, 충돌해도 오류 없이 서로의 값을 덮어쓴다
- `KEYS`는 전체 스캔 동안 Redis를 멈추므로 운영에서 사용할 수 없다. `SCAN`으로 조회하려면 키에 패턴이 있어야 한다
- 메모리 저장소이므로 TTL 없는 키가 쌓이면 그대로 메모리 고갈로 이어진다

## 규칙

**1. 구분자는 콜론(`:`)만 사용한다**

Redis의 관례이며 RedisInsight 등 도구가 콜론을 기준으로 계층을 표시한다.

**2. `앱:도메인:용도:식별자` 순으로 좁혀 쓴다**

```
reused : auth : verify : 1043
  │       │      │        └── 식별자
  │       │      └── 용도
  │       └── 도메인
  └── 앱 (Redis를 다른 앱과 공유할 경우 대비)
```

앞에서부터 좁혀져야 `SCAN reused:auth:*` 같은 조회가 성립한다.

**3. 모든 키에 TTL을 설정한다**

무기한으로 두어야 하는 키는 이 문서에 이유를 적는다.

**4. 키에 개인정보를 넣지 않는다**

```
❌ reused:auth:reset:user@example.com
✅ reused:auth:reset:1043
```

Redis 키는 `SLOWLOG`, `MONITOR`, 모니터링 대시보드, APM 트레이스에 원문 그대로 노출된다. 이메일을 키에 넣으면 개인정보가 로그 전반에 남아 `NFR-LOG-003`을 위반한다.

비밀번호 재설정처럼 미인증 요청이라 이메일만 가진 경우에도, 서버가 먼저 `user_identities`에서 `provider = 'LOCAL'`인 행의 `identity_id`를 조회해 그 값으로 키를 만든다. 계정이 없으면 키를 만들지 않고 `204`만 응답하므로 계정 존재 여부도 드러나지 않는다 (`NFR-AUTH-018`). 소셜 인증 수단도 이메일을 가질 수 있으므로 제공자 조건을 빼지 않는다 (ADR-016).

## 키 목록

| 키 | 타입 | TTL | 용도 | 근거 |
| --- | --- | --- | --- | --- |
| `reused:auth:refresh:{userId}:{tokenId}` | String | 14일 | Refresh Token 화이트리스트 | ADR-005 |
| `reused:auth:verify:{identityId}` | String | 10분 | 이메일 소유 확인 코드 | ADR-016 |
| `reused:auth:verify-try:{identityId}` | Counter | 10분 | 소유 확인 코드 검증 시도 | `NFR-AUTH-017` |
| `reused:auth:reset:{identityId}` | String | 10분 | 비밀번호 재설정 코드 | ADR-016 |
| `reused:auth:reset-try:{identityId}` | Counter | 10분 | 재설정 코드 검증 시도 | `NFR-AUTH-017` |
| `reused:auth:resend:{identityId}` | Counter | 1시간 | 재발송 횟수 (시간당 5회) | 정책 값 |
| `reused:auth:resend-gap:{identityId}` | String | 60초 | 재발송 간격 잠금 | 정책 값 |
| `reused:auth:login-fail:{identityId}` | Counter | 10분 | 로그인 실패 횟수 | `NFR-AUTH-019` |
| `reused:chatbot:rate:{userId}` | Counter | 1분 | 챗봇 호출 빈도 제한 | ADR-010 |
| `reused:trade:lock:{tradeId}` | String | 짧게 | 거래 상태 변경 분산 락 | ADR-010 |
| `reused:chat:room:{roomId}` | Pub/Sub 채널 | — | 채팅 메시지 전달 | ADR-002, ADR-010 |

`verify`·`verify-try`는 이메일이 등록된 모든 인증 수단에, `reset`·`reset-try`·`login-fail`은 `LOCAL` 인증 수단에만 쓴다. `resend`·`resend-gap`은 소유 확인과 재설정 발송이 함께 쓰므로, 소셜 인증 수단에서는 소유 확인 발송에만 쓰인다 (ADR-016).

## 설계 의도

**Refresh Token 키에 `userId`를 앞에 둔 이유** — 비밀번호 변경·재설정 시 해당 사용자의 토큰을 전부 폐기해야 한다 (`NFR-AUTH-016`). 이 순서라야 `SCAN reused:auth:refresh:{userId}:*`로 대상을 찾을 수 있다.

**시도 횟수 카운터의 TTL을 코드와 같게 맞춘 이유** — 코드가 만료되면 카운터도 함께 사라져, 별도 정리 없이 "코드 하나당 5회"가 성립한다.

**재발송 제한을 두 키로 나눈 이유** — 간격(60초)은 키의 존재 자체가 잠금이므로 값이 필요 없고, 총량(시간당 5회)은 세어야 하므로 카운터다.

**Pub/Sub 채널도 같은 규약을 쓰는 이유** — 채팅 서버가 별도 런타임(Node.js)이라 규약이 어긋나면 가장 먼저 깨진다.

## 운영 주의

- 운영 환경에서 `KEYS`를 사용하지 않는다. 조회가 필요하면 `SCAN`을 쓴다
- 로그인 실패 카운터는 계정이 없는 이메일로 시도할 경우 `identityId`를 만들 수 없다. 이때 카운트를 생략하든 다른 축으로 제한하든, **응답은 계정이 있을 때와 동일해야 한다** (`NFR-AUTH-018`)
- Redis는 진실의 원천이 아니다. 여기 있는 값이 모두 사라져도 재로그인·재발송으로 복구 가능해야 한다 (ADR-010)
