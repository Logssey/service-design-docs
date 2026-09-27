# 백엔드 DB 환경과 마이그레이션 전환

이 문서는 **현재 병합된 백엔드 코드**와 [마이그레이션 작업 #43](https://github.com/Logssey/service-backend/issues/43)의 목표를 구분한다. PostgreSQL은 회원·상품·거래·채팅·커뮤니티·신고 등의 진실의 원천이고, Redis는 토큰·인증 코드·속도 제한·채팅 Pub/Sub 등의 공유 상태다. Redis를 PostgreSQL의 대체 DB로 쓰지 않는다.

| 환경 | PostgreSQL·Redis | 데이터와 검증 |
| --- | --- | --- |
| PR CI | Docker 위 Testcontainers가 테스트 실행 중 각각 임시 컨테이너를 기동 | 스키마 001→002→003과 카테고리, 테스트별 픽스처를 넣고 실제 DB 쿼리·트랜잭션을 검증한다. 종료 후 테스트 데이터는 버린다. 별도 공유 DB·운영 비밀키가 필요 없다. |
| 로컬 웹 시연 | 개발자가 기동한 PostgreSQL·Redis. 테스트 컨테이너와 별개 | 현재 백엔드 README의 `docker run` 및 SQL 적용 절차를 따른다. 카테고리 외 회원·상품·채팅방은 회원가입/API 사용 또는 **로컬 전용** 시드가 있어야 보인다. 컨테이너를 재생성하면 이름 없는 저장 계층의 데이터가 사라질 수 있으므로 시연 데이터를 보존하려면 별도 볼륨·백업을 둔다. |
| 스테이징·운영 | 관리형 PostgreSQL(RDS 등)·Redis 및 실제 외부 연동 | 연결 설정과 시크릿만 환경별로 주입한다. 스키마·인증정보·백업·네트워크·TLS·배포 순서는 인프라 저장소에서 관리·검증한다. CI의 임시 데이터는 운영으로 복사하지 않는다. |

## 현재 스키마 적용과 #43 목표

현재 `develop`은 `spring.jpa.hibernate.ddl-auto=none`이고 `schema/001_init.sql`, `002_seed_categories.sql`, `003_profile_images.sql`을 사용한다. Testcontainers는 이 순서로 자동 초기화하지만, 로컬·운영 DB에는 백엔드 README에 따라 아직 수동 적용한다. 테스트가 통과했다는 사실만으로 이미 운영 중인 DB에 새 스키마가 적용되지는 않는다.

[#43](https://github.com/Logssey/service-backend/issues/43)의 목표는 **동일한 SQL 원본을 버전 관리형 마이그레이션으로 실행**해 새 빈 DB의 테스트·로컬·운영 경로가 달라지지 않게 하는 것이다. #43이 병합·검증되기 전에는 Flyway 자동 적용을 현재 동작이라고 설명하지 않는다. 이미 001~003을 수동 적용한 DB는 테이블·인덱스·데이터와 적용 순서를 먼저 확인한다. 자동 `baselineOnMigrate`를 켜거나 기존 스크립트를 재실행하지 않고, 백업 후 수동 baseline/전환 계획을 결정한다. 개발용 DB 재생성은 가능하지만 운영 데이터를 초기화하지 않는다.

스키마 변경은 기존 적용 파일을 고치지 않고 다음 버전 파일로 추가한다. DDL의 설계 원본은 [DB DDL](database-ddl.md)과 [003 프로필 이미지](profile-images-migration.md)에 있고, 실제 실행 파일은 백엔드 저장소 `schema/`가 소유한다. 마이그레이션 체계가 확정되면 백엔드 README의 수동 적용 절차와 이 문서를 함께 갱신한다.

## CI/CD와 클라우드 연결

백엔드 `Marketplace tests` 워크플로는 `develop`·`main` 대상 PR에서 `./gradlew test marketplaceE2E`를 실행한다. Spring 통합 테스트가 PostgreSQL·Redis Testcontainers를 직접 기동하므로 Actions에 고정 DB를 별도로 만들거나 운영 RDS 접속값을 넣을 필요가 없다. 카카오·SMTP·S3·LLM은 테스트 대역 또는 비활성 경로이며, 이 결과는 실제 클라우드 연결의 증명이 아니다.

이미지 빌드·배포용 `CI/CD` 워크플로는 `main` PR에 대해 테스트 제외 빌드와 ECR 이미지·GitOps 태그 갱신을 한다. **현행 워크플로에 RDS 스키마 적용 단계는 없다.** #43의 마이그레이션 전략을 정한 뒤 배포 전 작업(Job 또는 통제된 시작 절차), 실패 시 중단, 백업·복구를 GitOps/인프라 측에 연결해야 한다. 서비스 기동 시 자동 마이그레이션을 선택한다면 다중 인스턴스 경합, DB 권한, 이전 버전 앱과의 호환성을 먼저 검증한다.

로컬 → RDS·관리형 Redis 전환에서 도메인 API 코드를 바꾸는 것이 기본 전제는 아니다. 런타임의 `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`와 Redis 접속/인증/TLS 설정, 네트워크 접근을 환경별로 주입한다. 채팅 서버 `CHAT_REDIS_URL`은 API와 **같은** Redis를 가리키게 한다. CI에는 임시 DB용 설정만 쓰고, 운영 접속값·`JWT_SECRET`·외부 API 키는 Git이나 공개 문서가 아닌 CI/CD·런타임 시크릿 경로에 둔다. `.env` 파일 자체가 배포 산출물일 필요는 없으며 필요한 것은 해당 값의 안전한 주입이다.
