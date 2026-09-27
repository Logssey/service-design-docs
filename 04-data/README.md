# 04. 데이터 설계

데이터 모델과 실제 DDL을 분리해 관리한다. 영속 저장소는 PostgreSQL이며(ADR-009), Redis는 공유 상태 저장소로만 사용한다(ADR-010).

## 문서

- [스키마 및 ERD](schema-and-erd.md)
- [DB DDL](database-ddl.md)
- [백엔드 DB 환경과 마이그레이션 전환](backend-db-environments.md)
- [Redis 키 규약](redis-keys.md)

`database-ddl.md`는 설계 기준 DDL이다. 현재 실제 실행 스크립트는 백엔드 저장소 `schema/`가 소유한다. CI 테스트는 그 스크립트로 임시 DB를 초기화하며, Flyway 전환은 백엔드 #43이 병합되기 전까지 예정 작업이다.

## 관련 문서

- [비즈니스 규칙](../02-functional-design/business-rules.md)
- [서비스 아키텍처](../03-architecture/README.md)
- [API 명세](../05-api/README.md)
