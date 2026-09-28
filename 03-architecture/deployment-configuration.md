# 애플리케이션 배포 설정값

API 서버, 채팅 서버, 웹 프론트엔드가 실행·빌드에 요구하는 설정값을 한곳에 모은다. 이 문서는 **애플리케이션이 요구하는 값의 계약**이다. 값을 어디에 저장하고 어떻게 주입할지(Secret 관리 도구, 네트워크, Ingress, 매니페스트)는 인프라 docs 레포와 GitOps 레포가 소유한다.

기준 코드: 백엔드 `develop`(API `src/main/resources/application.properties`, 채팅 `chat-server/src/config.ts`), 프론트엔드 `develop`(`src/vite-env.d.ts`, `.github/workflows/ci.yml`). 코드와 이 문서가 다르면 코드가 기준이며, 이 문서를 고친다.

## 원칙

- 값은 **환경변수로 주입**한다. 설정 파일에는 플레이스홀더와 기본값만 둔다 (`NFR-CRED-001`, `NFR-CRED-007`, [워크로드 구성](workload-and-open-items.md)).
- 비밀값은 형상관리 저장소, 이미지, 로그, 관리자 화면에 두지 않는다 (`NFR-CRED-006`, `NFR-CRED-007`). `.env` 파일은 로컬 편의용이며 배포 산출물이 아니다 ([백엔드 DB 환경](../04-data/backend-db-environments.md)).
- API 서버와 채팅 서버는 **실행 시점**에 환경변수를 읽는다. 웹 프론트엔드의 `VITE_*` 값은 **빌드 시점**에 번들에 들어가므로 CI 빌드 단계에서 정해야 하고, 바꾸려면 재빌드한다. 번들은 브라우저로 전달되므로 `VITE_*`에 비밀값을 넣지 않는다.
- 아래 표의 **종류**는 주입 경로를 뜻한다. `Secret`은 비밀 저장소를 거쳐 주입하고, `Config`는 일반 설정으로 주입하며, `Build`는 CI 빌드 변수다.

## 배포 경로 (현재 CI 기준)

서비스 리전은 **도쿄(`ap-northeast-1`)** 다. ECR, S3 버킷, RDS, Redis를 같은 리전에 둔다.

| 워크로드 | 이미지 | 배포 값 위치 | CI 트리거 |
| --- | --- | --- | --- |
| API 서버 | ECR `logssey/reused-api` (`ap-northeast-1`) | GitOps `Logssey/gitops` `apps/reused-api/values.yaml` | `main` 대상 PR |
| 웹 프론트엔드 | ECR `logssey/reused-web` (`ap-northeast-1`) | GitOps `Logssey/gitops` `apps/reused-web/values.yaml` | `main`·`develop` 대상 PR. 이번 기능 PR은 lint·test·build도 수행 |
| 채팅 서버 | 백엔드 CI에 없음 | 백엔드 CI에 없음 | `tests.yml`이 검증용 이미지만 빌드한다 |

CI는 OIDC로 `secrets.AWS_ROLE_ARN`을 맡아 ECR에 올리고, GitHub App(`secrets.GITOPS_APP_ID`, `secrets.GITOPS_APP_PRIVATE_KEY`)으로 GitOps 레포의 이미지 태그만 갱신한다. **서비스 레포의 CI에는 RDS 스키마 적용 단계와 채팅 서버 배포 경로가 없다.** 인프라 쪽에서 따로 다루는지는 인프라 docs 레포에서 확인한다.

## 1. API 서버 (`reused-api`)

컨테이너 포트 `8080`. 상태 점검은 `/actuator/health/liveness`, `/actuator/health/readiness`다(`management.endpoint.health.probes.enabled=true`, 노출은 `health`만).

### 1.1 운영에서 넣어야 하는 값

| 변수 | 종류 | 필수 | 운영 값 | 로컬 값 |
| --- | --- | --- | --- | --- |
| `JWT_SECRET` | Secret | O | 32바이트(256비트) 이상 무작위 값 (api-spec §0.3). 기본값이 없어 설정하지 않으면 기동하지 않는다. 교체하면 발급된 Access Token이 모두 무효가 된다 | `local-dev-secret-key-must-be-at-least-32-bytes-long` |
| `SPRING_DATASOURCE_URL` | Config | O | RDS JDBC URL | `jdbc:postgresql://localhost:5432/reused` |
| `SPRING_DATASOURCE_USERNAME` | Secret | O | 애플리케이션 전용 DB 계정 (`NFR-CRED-002`) | `reused` |
| `SPRING_DATASOURCE_PASSWORD` | Secret | O | 위 계정의 비밀번호 | `reused` |
| `SPRING_DATA_REDIS_HOST` | Config | O | 관리형 Redis 주소. **채팅 서버와 같은 Redis** | `localhost` (기본값) |
| `SPRING_DATA_REDIS_PORT` | Config | | 기본 `6379` | — |
| `SPRING_DATA_REDIS_PASSWORD` | Secret | Redis 인증 시 | Redis 인증 값 | — |
| `SPRING_DATA_REDIS_SSL_ENABLED` | Config | TLS 사용 시 | `true` | — |
| `KAKAO_CLIENT_ID` | Secret | O | 카카오 REST API 키. 기본값이 없어 **설정하지 않으면 기동하지 않는다** | `local` 프로파일이면 불필요(기본 `stub`) |
| `KAKAO_CLIENT_SECRET` | Secret | 카카오 앱 설정 시 | 카카오 Client Secret | — |
| `MAIL_HOST` / `MAIL_PORT` | Config | O | 운영 SMTP. 기본값(`localhost`/`1025`)은 로컬 Mailpit용이다 | 기본값 |
| `MAIL_USERNAME` / `MAIL_PASSWORD` | Secret | SMTP 인증 시 | SMTP 자격증명 | — |
| `MAIL_FROM` | Config | | 발신 주소. 기본 `no-reply@reused.local`은 운영에 맞지 않는다 | 기본값 |
| `IMAGE_S3_BUCKET` | Config | 이미지 사용 시 O | 비공개 버킷 이름. 비어 있으면 서버는 뜨지만 이미지 API가 503이다 | `reused-images` |
| `IMAGE_S3_REGION` | Config | O | **`ap-northeast-1`(도쿄)**. 서비스 리전은 도쿄로 정했다. 코드 기본값은 `ap-northeast-2`(서울)라서 반드시 명시한다 | `us-east-1`(로컬 S3 대역) |
| `ANTHROPIC_API_KEY` | Secret | | 챗봇 자유 입력용. 비어 있으면 자유 입력 503 | — |
| `CHATBOT_FREE_INPUT_ENABLED` | Config | | 기본 `false`. 개인정보 외부 전송 정책 검토 전에는 켜지 않는다 | — |

### 1.2 운영에 넣지 않는 값

| 변수 | 이유 |
| --- | --- |
| `SPRING_PROFILES_ACTIVE=local` | 로컬 설정이 섞인다. 카카오 대역은 `local` 프로파일과 `KAKAO_STUB`가 모두 있어야만 켜지지만, 운영에는 넣지 않는다 |
| `KAKAO_STUB` | `local` 프로파일 전용 |
| `APP_AUTH_COOKIE_SECURE=false` | 기본값 `true`(HTTPS 전용 쿠키)가 운영 값이다 |
| `IMAGE_S3_ENDPOINT` | LocalStack·MinIO 같은 로컬 S3 대역용. AWS에서는 비운다 |
| `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN` | 정적 키 대신 워크로드 역할을 쓴다. 필요 권한은 대상 버킷의 `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` |

### 1.3 기본값을 두는 값 (필요할 때만 바꾼다)

| 변수 | 기본값 | 설명 |
| --- | --- | --- |
| `ANTHROPIC_BASE_URL` | SDK 기본 주소 | 게이트웨이를 거칠 때만 |
| `CHATBOT_ENABLED` | `true` | 환경 강제 차단. `false`면 관리자 DB 스위치를 켜도 두 챗봇 엔드포인트가 503 |
| `IMAGE_ORPHAN_RETENTION` | `24h` | 게시글에 연결되지 않은 이미지 보존 기간 |
| `IMAGE_CLEANUP_INTERVAL` | `1h` | 고아 이미지 정리 주기 |
| `IMAGE_UNATTACHED_LIMIT` | `20` | 사용자별 미연결 이미지 상한 |

`application.properties`의 정책 값(`app.auth.access-token-ttl=30m`, `app.auth.refresh-token-ttl=14d`, `app.auth.local.*`, `app.moderation.*`, `app.chatbot.rate-limit` 등)도 Spring 규칙에 따라 환경변수로 덮어쓸 수 있다(예: `APP_AUTH_ACCESS_TOKEN_TTL`). 규칙 명세의 정책 값과 같으므로 운영에서 바꾸지 않는 것을 원칙으로 하고, 바꾼다면 [비즈니스 규칙](../02-functional-design/business-rules.md)을 먼저 고친다.

### 1.4 실행 조건

- **스키마:** 이번 백엔드 PR의 Flyway는 기동 시 `V1`~`V5`(`schema/001`~`005`)를 적용한다. `V5`는 관리자 챗봇 스위치 테이블을 만든다. 비어 있지 않은 기존 DB는 자동 기준선을 쓰지 않으므로(`spring.flyway.baseline-on-migrate=false`) `flyway_schema_history` 없이 기동하면 안전하게 실패한다. 담당자가 실제 스키마를 확인하고 적용된 마지막 버전으로 수동 baseline을 기록한 뒤 기동한다. 절차는 백엔드 README "DB 스키마와 마이그레이션"과 [백엔드 DB 환경](../04-data/backend-db-environments.md)에 있다. `V4`의 적용 전 점검은 [004 마이그레이션](../04-data/social-identity-email-migration.md)에 있다.
- **다중 인스턴스:** 예약 작업(고아 이미지 정리, 정지 만료 해제)은 행 잠금(`FOR UPDATE SKIP LOCKED`)과 행 단위 조건부 갱신으로 여러 인스턴스에서 돌아도 중복 처리하지 않는다.
- **같은 도메인:** API 서버에는 CORS 설정이 없다. Refresh 쿠키도 `Path=/api/v1/auth`, `SameSite=Lax`다. 따라서 브라우저는 웹과 **같은 도메인의 `/api` 경로**로 API를 호출해야 한다. 도메인을 나누려면 CORS와 쿠키 정책을 함께 다시 설계한다.

## 2. 채팅 서버 (`reused-chat`, 배포 경로 미정)

컨테이너 포트는 `PORT`(기본 `3001`)다. JWT 서명 키를 갖지 않고, 받은 Access Token을 API 서버 엔드포인트(`GET /api/v1/chat/session`, `GET /api/v1/chat-rooms/{id}/subscription`)에 그대로 넘겨 계정 상태와 방 참여 권한을 확인한다. 그래서 채팅 서버에서 API 서버로 가는 네트워크 경로가 필요하다. 값이 조건에 맞지 않으면 기동 시 실패한다(`chat-server/src/config.ts`).

| 변수 | 종류 | 필수 | 운영 값 | 로컬 값 |
| --- | --- | --- | --- | --- |
| `NODE_ENV` | Config | O | `production`. 이때 `CHAT_ALLOWED_ORIGINS`가 필수가 된다 | 미설정(`development`) |
| `CHAT_API_BASE_URL` | Config | O | API 서버의 클러스터 내부 주소(예: `http://reused-api:8080`). `http`/`https`만, 자격증명·쿼리 금지 | `http://localhost:8080` |
| `CHAT_REDIS_URL` | Secret | O | API 서버와 **같은** Redis. TLS면 `rediss://`. 비밀번호가 URL에 들어가므로 Secret으로 둔다 | `redis://localhost:6379` |
| `CHAT_ALLOWED_ORIGINS` | Config | 운영 O | 웹 도메인(예: `https://{도메인}`). 쉼표 구분, 와일드카드 금지 | `http://localhost:5173` |
| `PORT` / `HOST` | Config | | 기본 `3001` / `0.0.0.0` | 기본값 |

조정 값(기본값 유지 권장): `CHAT_REQUEST_TIMEOUT_MS`(2000), `CHAT_MAX_TOKEN_LENGTH`(8192), `CHAT_MAX_SUBSCRIPTIONS`(50), `CHAT_MAX_CLIENT_PAYLOAD_BYTES`(16384), `CHAT_MAX_REDIS_PAYLOAD_BYTES`(65536), `CHAT_DELIVERY_AUTH_CONCURRENCY`(50), `CHAT_DELIVERY_AUTH_QUEUE_LIMIT`(2000), `CHAT_SHUTDOWN_TIMEOUT_MS`(10000). 범위는 채팅 서버 README에 있다.

## 3. 웹 프론트엔드 (`reused-web`)

nginx(비루트, 포트 `8080`)가 빌드 산출물을 서빙하며 `/api`를 프록시하지 않는다. `/api` 경로는 Ingress가 API 서버로 보낸다(1.4 같은 도메인 조건).

| 변수 | 종류 | 운영 값 | 코드 기본값(미설정 시) | 현재 CI |
| --- | --- | --- | --- | --- |
| `VITE_API_BASE_URL` | Build | `/api/v1` | `/api/v1` | `vars.VITE_API_BASE_URL` 또는 `/api/v1` |
| `VITE_USE_MOCKS` | Build | `false` | **목 켜짐**(`'false'`가 아니면 모두 목) | `vars.VITE_USE_MOCKS` 또는 `false` |
| `VITE_USE_MOCKS_AUTH` | Build | 미설정 | `VITE_USE_MOCKS`를 따름 | 미설정 |
| `VITE_KAKAO_STUB` | Build | 미설정 또는 `false` | 목 모드일 때만 켜짐 | 미설정 |
| `VITE_KAKAO_CLIENT_ID` | Build | 카카오 REST API 키(인가 요청 주소에 드러나는 공개 값) | **빈 문자열** | `vars.VITE_KAKAO_CLIENT_ID`; 비어 있으면 경고 |
| `VITE_KAKAO_REDIRECT_URI` | Build | 미설정 권장 | 접속 도메인 + `/oauth/callback` | `vars.VITE_KAKAO_REDIRECT_URI` 또는 빈 문자열 |
| `VITE_CHAT_REALTIME` | Build | 게이트웨이 배포·Ingress 준비 후 `true` | `false` | `vars.VITE_CHAT_REALTIME` 또는 `false` |
| `VITE_CHAT_SOCKET_URL` | Build | 별도 게이트웨이 도메인을 쓸 때만 설정 | 같은 도메인의 `/socket.io` | `vars.VITE_CHAT_SOCKET_URL` 또는 빈 문자열 |
| `VITE_API_PROXY_TARGET` | — | 해당 없음 | `http://localhost:8080` | 개발 서버 전용 |
| `VITE_CHAT_PROXY_TARGET` | — | 해당 없음 | `http://localhost:3001` | 개발 서버 전용 |

- `VITE_KAKAO_CLIENT_ID`가 비면 카카오 인가 요청이 빈 `client_id`로 나가 로그인이 실패한다. 공개 값이므로 CI에서는 `secrets`가 아니라 저장소 Variables(`vars.VITE_KAKAO_CLIENT_ID`)로 넣는다.
- 리디렉트 URI는 기본값(접속 도메인 기준)을 쓰고, 그 주소를 카카오 개발자 콘솔에 등록한다.

## 4. 외부에서 발급·준비할 것

| 대상 | 필요한 것 | 쓰는 곳 |
| --- | --- | --- |
| 카카오 개발자 앱 | REST API 키, Client Secret(설정 시), 운영 리디렉트 URI `https://{도메인}/oauth/callback` 등록 | `KAKAO_CLIENT_ID`, `KAKAO_CLIENT_SECRET`, `VITE_KAKAO_CLIENT_ID` |
| 메일 발송 수단 | SMTP 호스트·포트·계정, 발신 주소 도메인 인증 | `MAIL_*` |
| S3 버킷 | 공개 차단·암호화, 웹 도메인의 `PUT`·`Content-Type` CORS 허용, `pending/` 1일 Lifecycle | `IMAGE_S3_*` |
| 워크로드 역할 | 위 버킷의 `s3:GetObject`·`PutObject`·`DeleteObject` | API 서버 파드 |
| RDS | 애플리케이션 전용 계정, `V1`~`V5` 적용 또는 수동 baseline | `SPRING_DATASOURCE_*` |
| 관리형 Redis | API·채팅 공용 인스턴스, 인증·TLS 설정 | `SPRING_DATA_REDIS_*`, `CHAT_REDIS_URL` |
| LLM API 키 (선택) | 챗봇 자유 입력을 켤 때만 | `ANTHROPIC_API_KEY` |

S3 운영 조건의 상세는 백엔드 README "이미지 저장소 운영 조건"에 있다.

## 5. 배포 순서

1. 비밀값과 설정값을 준비한다(1~4장).
2. DB 스키마를 맞춘다. 새 DB는 Flyway가 기동 시 적용하고, 기존 DB는 수동 baseline을 기록한다(1.4).
3. API 서버를 배포하고 readiness를 확인한다.
4. 채팅 서버를 배포한다(배포 경로가 정해진 뒤).
5. 웹 프론트엔드를 배포한다. 새 API 필드를 쓰는 화면이 있으므로 API 서버보다 먼저 내보내지 않는다.
6. 운영에서 한 번씩 확인한다: 카카오 로그인, 인증 메일 수신, 이미지 업로드·조회, 채팅 실시간 수신, 챗봇 추천 질문.

## 6. 미결정·확인 필요

| 항목 | 내용 | 후속 |
| --- | --- | --- |
| 채팅 서버 배포 경로 | ECR 저장소·GitOps values·CI 작업이 없다 | 백엔드 CI, GitOps 레포 |
| 운영 메일 발송 수단 | 미결정 (ADR-016 결과) | 클라우드 아키텍처 |
| 카카오 실제 로그인 | CI는 공개 키 변수를 번들에 넣지만, GitHub Variables 값과 카카오 운영 리디렉트 URI 등록은 별도 필요 | 프론트엔드 CI·카카오 콘솔 |
| RDS 스키마 적용 단계 | CI/CD에 없다. Flyway 기동 적용과 수동 baseline 중 운영 방식 결정 | [백엔드 DB 환경](../04-data/backend-db-environments.md) |
| 이미지 버킷 리전 기본값 | 서비스 리전은 도쿄(`ap-northeast-1`)로 정했지만 코드의 `IMAGE_S3_REGION` 기본값은 `ap-northeast-2`다. 운영에서는 값을 명시하고, 기본값도 맞출지 정한다 | 백엔드 `application.properties`, README |
| GitOps values 내용 | 이 문서의 변수가 `apps/reused-api`, `apps/reused-web` values에 빠짐없이 있는지 대조하지 않았다 | GitOps 레포 |
