# 백엔드 구현 상태

기준: 2026-09-28, `Logssey/service-backend`의 `develop` (`e243e22`, PR #45–#49 병합) 및 이번 `feat/finish-marketplace-api` 작업 브랜치. 아래 3개 신규 관리자 API는 **작업 브랜치의 변경**이므로 백엔드 PR 병합 전 `develop`에 있다고 해석하면 안 된다. 상품 상태 필터는 [PR #49](https://github.com/Logssey/service-backend/pull/49)에도 반영되었다. `Yes`는 명세된 API 동작이 코드·자동 테스트로 검증되었다는 뜻이며, 화면 연결이나 클라우드 배포가 완료되었다는 뜻은 아니다.

## API 현황

카탈로그 88개 중 **87개 Yes, 1개 No**. 기존 85개 중 콘텐츠 의존 경로 5개(`POST /reports`, `GET /admin/dashboard`, `GET /admin/reports`, `PATCH /admin/reports/{reportId}`, `GET /admin/users`)는 [PR #48](https://github.com/Logssey/service-backend/pull/48)에서 실제 상품·커뮤니티·거래 데이터 및 관리자 조치와 연결하고 통합 테스트를 추가했다. 이번 작업은 `GET/PATCH /admin/chatbot`과 `GET /admin/credentials/status`를 추가한다. `GET /listings`의 `itemCondition` 필터는 PR #49와 이번 작업 브랜치에 모두 구현되어 병합으로 단일화했다. URL에는 `/api/v1` 접두사를 생략했다.

남은 `No`는 `POST /chatbot/messages`의 **자유 입력 모드**다. 추천 질문 ID를 보낸 응답은 사용할 수 있다. 자유 입력은 개인정보가 외부 LLM으로 전달될 위험에 대해 사용자와 합의한 결정에 따라 기본 503으로 차단한다. API 경로와 구현체가 있다는 사실만으로 해당 기능을 완료로 표시하지 않는다.

`FR-AI-008`은 관리자 상태 조회·변경 API와 화면으로 다루고, 공유 DB의 스위치를 모든 API 인스턴스가 읽는다. `CHATBOT_ENABLED=false`는 관리자도 해제할 수 없는 상위 차단이다. `FR-CRED-005`는 관리자에게 설정 존재·사용 여부만 보여 준다. 값·일부·지문·접속 URL·사용자명은 반환하지 않는다. 이 조회는 실제 연동 성공 검사가 아니다.

## 구현된 흐름과 검증

- 회원 인증(이메일·카카오), 상품·카테고리·이미지, 관심 상품, 거래, REST·Socket.IO 채팅, 후기, 판매자·내 프로필, 커뮤니티, 신고·차단, 알림, 공지, 관리자 상품·거래·회원·신고·공지·대시보드·감사 API를 구현했다.
- Flyway가 **새 빈 DB**에 `V1` 초기 스키마 → `V2` 카테고리 → `V3` 프로필 이미지 → `V4` 소셜 선택 이메일 → `V5` 챗봇 운영 설정을 적용한다. PostgreSQL·Redis Testcontainers도 운영 JAR와 같은 마이그레이션을 실행한다. `APP_DEMO_SEED=true`와 `local-demo` 프로필을 함께 켠 별도 `reused_demo_*` DB에만 가상 계정·상품·거래·커뮤니티 데이터를 넣는다.
- 2026-09-28 백엔드 작업 브랜치에서 전체 `./gradlew.bat test --no-daemon` 실행 결과 **74개 스위트, 928개 테스트 통과**, 실패·오류·건너뜀 0. 이 수치는 PR #49를 병합하기 전 실행이므로 병합 후 영향받는 통합 테스트와 백엔드 PR CI도 확인한다. 이전 PR CI에서는 별도 `marketplaceE2E`, Node 채팅 테스트 및 채팅 Docker 이미지 빌드가 통과했다.
- 테스트의 카카오·SMTP·S3는 대역을 사용한다. 실제 클라우드 자격증명·네트워크·운영 데이터·브라우저 화면 연결까지 검증한 결과는 아니다.

## 운영 및 제품 완료까지 남은 일

1. 이번 프론트 작업 브랜치에서 신고→관리자 조치, 알림·공지·판매자·계정 화면과 실시간 채팅을 연결했다. 로컬 시드 DB·Moto S3·채팅 게이트웨이의 주요 화면 스모크는 통과했지만 운영 환경의 전체 브라우저 E2E와 [PRD 릴리스 조건](../01-requirements/product-requirements.md)은 별도 확인한다.
2. 이미지 버킷의 비공개 접근, 권한, CORS, `pending/` Lifecycle과 실제 S3 경로를 검증한다. `IMAGE_S3_BUCKET`이 비어 있으면 이미지 API는 503이다. 실제 SMTP·카카오 설정도 별도 검증한다.
3. 채팅 런타임을 API와 별도로 운영 배포하고 `/socket.io` WebSocket 업그레이드·Origin·Redis 연결을 확인한다. 현재 CI/CD의 Spring 이미지 배포만으로 채팅은 배포되지 않는다.
4. **기존 데이터가 있는 DB**는 스키마·백업·복구 가능성을 확인한 뒤 수동 Flyway baseline을 결정한다. 자동 baseline은 꺼져 있다. [DB 환경 가이드](../04-data/backend-db-environments.md)를 따른다.
5. `FR-CRED-004/006`의 승인된 운영 권한·시크릿 교체 반영, 비기능 요구사항의 성능·복구·관측성 및 운영 시크릿·배포 검증 결과를 문서화한다. 관리자 상태 조회만으로 자격증명 교체·연동 검증이 완료되지는 않는다.
