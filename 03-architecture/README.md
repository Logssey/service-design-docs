# 03. 서비스 아키텍처 결정 기록

> 서비스 애플리케이션 계층의 현재 아키텍처 결정이다. 클라우드 인프라·배포·CI/CD 상세 설계는 인프라 docs 레포에서 관리한다.

## 상태 정의

| 상태 | 의미 |
| --- | --- |
| `Accepted` | 구현 기준으로 승인됨 |
| `Proposed` | 결정 대기. 담당자 지정 후 확정 필요 |
| `Superseded` | 후속 ADR로 대체됨 |

## 결정 목록

| ID | 결정 | 상태 |
| --- | --- | --- |
| [ADR-001](adr/ADR-001.md) | 백엔드 프레임워크로 Spring Boot 채택 | Accepted |
| [ADR-002](adr/ADR-002.md) | 채팅 서버를 독립 워크로드로 분리 | Accepted |
| [ADR-003](adr/ADR-003.md) | 챗봇을 API 서버에 통합하고 어댑터로 경계 분리 | Accepted |
| [ADR-004](adr/ADR-004.md) | 소셜 로그인 제공자를 Kakao 단독으로 한정 | Accepted |
| [ADR-005](adr/ADR-005.md) | JWT 기반 인증과 Refresh Token 화이트리스트 | Accepted |
| [ADR-006](adr/ADR-006.md) | 역할과 소유권의 2단 접근 제어 | Accepted |
| [ADR-007](adr/ADR-007.md) | 거래를 독립 엔티티로 모델링 | Accepted |
| [ADR-008](adr/ADR-008.md) | 거래 완료 확정 주체를 구매자로 한정 | Accepted |
| [ADR-009](adr/ADR-009.md) | PostgreSQL을 단일 데이터 저장소로 사용 | Accepted |
| [ADR-010](adr/ADR-010.md) | Redis를 공유 상태 저장소로 도입 | Accepted |
| [ADR-011](adr/ADR-011.md) | 이미지 저장에 S3와 Presigned URL 사용 | Accepted |
| [ADR-012](adr/ADR-012.md) | 탈퇴 시 논리 삭제와 작성자 익명화 | Accepted |
| [ADR-013](adr/ADR-013.md) | 커서 기반 페이지네이션 | Accepted |
| [ADR-014](adr/ADR-014.md) | 인앱 알림 전용, 폴링 기반 조회 | Accepted |
| [ADR-015](adr/ADR-015.md) | 프론트엔드 프레임워크 선택 | Accepted |
| [ADR-016](adr/ADR-016.md) | 자체 이메일·비밀번호 계정을 소셜 로그인과 병행 | Accepted |
| [ADR-017](adr/ADR-017.md) | 인증 수단을 계정과 분리된 엔티티로 모델링 | Accepted |
| [ADR-018](adr/ADR-018.md) | 탈퇴 시 인증 수단 파기, 재가입 제한 없음 | Accepted |

## 후속 검토

- [결정에 따른 워크로드 구성, 후속 문서 이관 및 미결정 사항](workload-and-open-items.md)

초기 ADR은 현재 결정과 번호가 중복되고 내용이 변경되었으므로 [초기 설계 문서](../archive/initial-design/README.md)에만 보관한다.
