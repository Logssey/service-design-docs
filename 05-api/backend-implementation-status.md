# 백엔드 구현 상태

기준: 2026-09-27, `Logssey/service-backend`의 `develop` (`e18d9f7`, [PR #41](https://github.com/Logssey/service-backend/pull/41) 병합). 이 문서는 **현재 병합된 코드**와 후속 작업을 구분한다. 카탈로그의 `개발완료여부: Yes`는 명세된 동작이 코드·자동 테스트로 검증된 경우이며 운영 배포·클라우드 연동 완료를 뜻하지 않는다.

## 엔드포인트 현황

카탈로그 85개 중 **79개 Yes, 6개 No**. PR #41에서 기존 미구현 17개 모두 HTTP 경로와 서비스 코드가 추가되었지만, 다른 도메인의 실제 데이터를 연결하지 못한 경로를 완료로 세지 않는다. 기존 68개에 비밀번호 변경, 내 신고 목록, 추천 질문 목록, 감사 로그 조회, 공지사항 관리·공개 조회 5개, 관리자 회원 역할·상태 변경 2개를 더해 79개다.

| 아직 No인 경로 | 현재 `develop` 동작 | 완료 조건 |
| --- | --- | --- |
| `POST /reports` | 회원 대상만 신고 가능. 상품·메시지·커뮤니티 게시글·댓글은 대상 조회가 없어 404 | [#42](https://github.com/Logssey/service-backend/issues/42): 실제 대상 조회와 권한·소유자 검증 |
| `POST /chatbot/messages` | 추천 질문 ID 답변은 가능. 자유 입력은 외부 LLM 경로가 있으나 개인정보 전송 위험 때문에 운영에서 비활성 유지 결정 | [#44](https://github.com/Logssey/service-backend/issues/44): 자유 입력 차단을 코드·테스트로 고정. 전체 명세 완료는 개인정보 정책 확정 뒤 별도 판단 |
| `GET /admin/dashboard` | 회원·신고 값은 실제 DB, 상품·완료 거래 값은 0 대역 | [#42](https://github.com/Logssey/service-backend/issues/42): 실제 집계 연결 |
| `GET /admin/reports` | 조회·필터는 동작. 회원 외 신고 대상 요약은 실제 콘텐츠와 연결되지 않음 | [#42](https://github.com/Logssey/service-backend/issues/42): 대상 유형별 실제 요약 연결 |
| `PATCH /admin/reports/{reportId}` | 신고 상태·회원 정지는 가능. 상품 숨김·삭제 및 커뮤니티 숨김 조치는 503 | [#42](https://github.com/Logssey/service-backend/issues/42): 콘텐츠 조치·감사·롤백 연결 |
| `GET /admin/users` | 회원 조회·필터는 동작. `listingCount`가 0 대역 | [#42](https://github.com/Logssey/service-backend/issues/42): 실제 상품 수 연결 |

위 URL은 모두 `/api/v1` 접두사를 생략했다. #42가 병합되고 실제 PostgreSQL·Redis 통합 검증까지 통과하면 위 5개 콘텐츠 의존 경로를 Yes로 변경할 수 있다. **그 전에는 84/85 또는 85/85로 표시하지 않는다.** 챗봇 자유 입력은 사용자의 비활성 유지 결정에 따라 #42와 별도로 No를 유지한다. 추천 질문 조회 및 ID 선택 답변은 계속 제공한다.

## 이미 구현한 거래 흐름

CATEGORIES, IMAGES(LISTING·PROFILE), LISTINGS, WISHES, TRADES, CHAT(REST·Socket.IO), REVIEWS, ME, 관리자 상품·거래, 회원 인증(이메일·카카오), 차단·알림·감사 저장, 커뮤니티 게시글·댓글 API가 있다. 커뮤니티는 공개 조회, 카테고리·최신순 커서, 차단 회원 필터, 탈퇴 작성자 익명화, 작성자 권한 및 댓글 수 갱신을 포함한다. 프론트 화면 또는 HTTP 경로 존재만으로 통합 완료를 판단하지 않는다.

## 검증 및 배포 경계

- `develop` PR #41의 CI는 PostgreSQL·Redis Testcontainers에서 Spring 테스트 861개와 실제 HTTP·Socket.IO 시장 흐름 E2E를 통과했다. 다만 신고 대상, 콘텐츠 조치, 관리자 상품 집계는 테스트 대역을 썼으므로 위 6개를 완료로 세지 않는다.
- [#42](https://github.com/Logssey/service-backend/issues/42)는 이 대역을 실제 도메인 어댑터로 교체하고 신고→조치→조회·복구까지 별도 통합 테스트로 검증하는 작업이다. 병합·CI 확인 전까지는 이 문서의 상태를 올리지 않는다.
- CI의 임시 PostgreSQL·Redis는 각 테스트 실행을 위한 컨테이너다. 현재 스키마는 `schema/001_init.sql` → `002_seed_categories.sql` → `003_profile_images.sql`을 Testcontainers 초기화에서 순서대로 적용한다. [#43](https://github.com/Logssey/service-backend/issues/43)의 마이그레이션 체계 전환은 아직 미병합이며 [DB 환경·전환 가이드](../04-data/backend-db-environments.md)에서 별도로 다룬다.
- 외부 카카오·SMTP·S3·LLM은 테스트에서 대역을 사용한다. 클라우드 자격증명·운영 경로, 운영 RDS·Redis와 이미 존재하는 DB의 마이그레이션은 별도 배포 검증이 필요하다.
- 테스트 명령: backend `./gradlew test marketplaceE2E`, `chat-server`의 `npm ci && npm run build && npm test`. GitHub Actions의 `Marketplace tests`는 `develop`·`main` 대상 PR에서 실행한다. 이미지 배포용 `CI/CD` 워크플로는 별도이며 테스트를 제외한 빌드를 수행하므로 그 성공만으로 API 검증을 대체할 수 없다.

## 운영 전 남은 결정

- 챗봇 자유 입력은 개인정보가 외부 LLM으로 전달될 가능성이 있어 **비활성 유지**한다. `NFR-EXT-004` 충족을 보장할 수 있기 전에는 API 키를 넣었다는 이유만으로 켜지 않는다. [챗봇 메시지 명세](<endpoints/chatbot/챗봇 메시지 전송.md>)에 현재 제한을 적었다.
- `FR-AI-008`의 “관리자가 챗봇 활성·비활성 상태를 관리”하는 방식은 85개 엔드포인트 카탈로그에 관리자 전용 API가 없다. 현재는 서버 환경설정 `CHATBOT_ENABLED`를 변경하고 재기동하는 운영 스위치다. 관리자 화면/API가 필요한지 별도 결정 전에는 해당 요구사항을 완료로 주장하지 않는다.
- 운영 DB에 이미 001~003이 수동 적용된 경우 [#43](https://github.com/Logssey/service-backend/issues/43) 전환 시 자동 baseline을 켜지 않는다. 데이터·스키마·적용 이력을 확인한 뒤 수동 baseline 또는 별도 마이그레이션 계획으로 이관한다.
