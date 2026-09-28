# 04. 데이터 설계

데이터 모델과 실제 DDL을 분리해 관리한다. 영속 저장소는 PostgreSQL이며(ADR-009), Redis는 공유 상태 저장소로만 사용한다(ADR-010).

## 문서

- [스키마 및 ERD](schema-and-erd.md)
- [DB DDL](database-ddl.md)
  - [003 프로필 이미지 연결](profile-images-migration.md)
  - [004 소셜 인증 수단 이메일](social-identity-email-migration.md)
- [백엔드 DB 환경과 마이그레이션 전환](backend-db-environments.md)
- [Redis 키 규약](redis-keys.md)

`database-ddl.md`는 설계 기준 DDL이다. 실제 실행 SQL은 백엔드 저장소 `schema/`가 소유하고, 새 빈 DB와 CI 임시 DB에는 Flyway `V1`~`V5`가 자동 적용된다(`V5`는 이번 백엔드 작업 브랜치). 기존 데이터가 있는 DB의 안전한 전환은 [DB 환경 가이드](backend-db-environments.md)를 따른다.

## 관련 문서

- [비즈니스 규칙](../02-functional-design/business-rules.md)
- [서비스 아키텍처](../03-architecture/README.md)
- [API 명세](../05-api/README.md)
