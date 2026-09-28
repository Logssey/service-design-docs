# 백엔드 DB 환경과 마이그레이션

기준: 2026-09-28, 백엔드 `develop` (`42f0362`)과 이번 `feat/finish-marketplace-api` 작업 브랜치. `V5`는 작업 브랜치 변경이라 PR 병합 전 `develop`에는 없다. PostgreSQL은 회원·상품·거래·커뮤니티·신고의 영속 데이터 저장소이고, Redis는 토큰·인증 코드·속도 제한·채팅 Pub/Sub 등 공유 상태다. Redis는 PostgreSQL을 대체하지 않는다.

| 환경 | DB와 데이터 | 검증 범위 |
| --- | --- | --- |
| PR CI | Testcontainers가 실행마다 독립 PostgreSQL·Redis를 기동하고 종료 시 버린다 | 운영 JAR와 같은 Flyway 마이그레이션, API 통합 테스트. 실제 RDS·Redis 접속 정보는 쓰지 않는다. |
| 로컬 시연 | 운영·기존 개발 DB와 다른 `reused_demo_*` PostgreSQL DB 및 Redis. 명시적 `local-demo` 프로필과 `APP_DEMO_SEED=true`에서만 가상 데이터 생성 | 화면·API 확인. 데이터 보존이 필요하면 전용 Docker 볼륨을 사용한다. [백엔드 데모 가이드](https://github.com/Logssey/service-backend/blob/develop/scripts/README-demo.md) 참고. |
| 스테이징·운영 | 관리형 PostgreSQL(RDS 등)·Redis와 실제 외부 저장소·메일 | 시크릿·네트워크·TLS·백업·복구 및 스키마 이력을 별도로 검증한다. CI/로컬 데이터를 복사하지 않는다. |

## 현재 Flyway 동작

백엔드는 `schema/001_init.sql`부터 `005_chatbot_feature_flag.sql`까지 하나의 SQL 원본을 유지하고, Gradle `processResources`가 각각 `V1`~`V5` Flyway 리소스로 패키징한다. **새 빈 DB**에서 앱이 시작될 때 순서대로 적용된다. Hibernate는 스키마를 만들지 않는다(`ddl-auto=none`). 앞으로의 스키마 변경은 기존 SQL을 수정하지 않고 다음 번호 파일을 `schema/`에 추가해 Gradle 매핑에도 등록한다.

기존 데이터가 있는 DB에 `flyway_schema_history`가 없으면 기동이 실패하도록 `baseline-on-migrate=false`를 유지한다. 자동 baseline이나 기존 SQL 재실행으로 운영 데이터에 손대지 않는다. 기존 DB 전환 담당자는 다음 순서를 따른다.

1. 대상 서버·DB명·스키마를 읽기 전용으로 확인하고 백업 및 복구 연습을 완료한다.
2. 실제 테이블·인덱스·제약과 `schema/` 원본을 비교한다. 특히 `V3` 프로필 이미지 컬럼, `V4` 소셜 선택 이메일 컬럼·제약과 `V5` 운영 스위치 테이블의 존재 여부를 확인한다. 일부만 적용되었거나 차이가 있으면 중단하고 별도 수정 계획을 세운다.
3. 정확히 적용된 최종 버전까지만 Flyway CLI로 **수동 baseline**한다. 예를 들어 `V4`까지 동일하고 `V5` 테이블이 없는 DB라면 baseline version `4`를 사용하고, 이후 `V5`를 적용한다. `V5`까지 동일한 DB에만 version `5` baseline을 고려한다. baseline은 SQL 내용 자체를 검증하거나 재실행하지 않으므로 비교·백업이 선행되어야 한다.
4. `flyway info`와 다음 마이그레이션을 확인하고, 운영과 같은 상태의 일회용 복제 DB에서 먼저 기동·롤백을 연습한다. 기존 버전 앱과 호환성이 필요한 DDL은 새 앱보다 먼저 적용하는 배포 순서를 정한다.

[백엔드 README](https://github.com/Logssey/service-backend/blob/develop/README.md#2-db-스키마와-마이그레이션)에 버전별 확인점과 명령 예시가 있다. 새 마이그레이션이 추가되면 이 문서의 버전 표기도 함께 갱신한다.

## CI/CD와 클라우드 연결

`Marketplace tests` 워크플로는 임시 DB에서 Spring 테스트와 별도 전체 거래 E2E를 실행한다. 이미지 빌드·배포용 `CI/CD` 워크플로는 Spring API 이미지를 ECR/GitOps로 전달한다. 운영 RDS의 기존 스키마에 대한 수동 baseline, 백업 및 복구 계획을 이 테스트가 대신하지 않는다. 새 빈 DB라면 앱 기동 시 Flyway가 적용되지만, 다중 인스턴스 시작·DDL 권한·이전 버전 호환성을 운영 환경에서 확인해야 한다.

로컬에서 RDS·관리형 Redis로 옮길 때 도메인 API 코드를 바꾸는 것이 기본 전제는 아니다. 환경별 PostgreSQL JDBC URL·계정·비밀번호 및 Redis 주소·인증·TLS, 네트워크 접근을 런타임에 주입한다. 별도 채팅 서버의 `CHAT_REDIS_URL`은 API와 같은 Redis를 가리킨다. 시크릿은 Git·공개 문서나 프론트 번들에 넣지 않고 CI/CD·런타임 비밀값 전달 경로로 관리한다.
