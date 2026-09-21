# 04. 데이터 설계

데이터 모델과 실제 DDL을 분리해 관리한다.

## 문서

- [스키마 및 ERD](schema-and-erd.md)
- [DB DDL](database-ddl.md)

`database-ddl.md`는 제공된 설계 기준 DDL이다. 백엔드 마이그레이션 체계가 확정되면 실제 실행 원본은 백엔드 저장소가 소유하고, 이 저장소에서는 설계 설명과 링크만 관리하는 것이 적합하다.

## 관련 문서

- [비즈니스 규칙](../02-functional-design/business-rules.md)
- [서비스 아키텍처](../03-architecture/README.md)
- [API 명세](../05-api/README.md)
