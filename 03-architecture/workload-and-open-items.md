# 결정에 따른 워크로드 구성

| **워크로드** | **기술** | **비고** |
| --- | --- | --- |
| API 서버 | Spring Boot | 챗봇 모듈 포함, 복수 인스턴스 |
| 채팅 서버 | Node.js + Socket.IO | 복수 인스턴스 |
| 프론트엔드 | React + Vite | 정적 파일, ADR-015 Accepted |
| 데이터베이스 | PostgreSQL |  |
| 캐시·Pub/Sub | Redis |  |
| 객체 스토리지 | S3 |  |

## 로컬 개발 구성

운영 환경의 인증정보 주입과 메일 발송 수단은 클라우드 아키텍처 문서가 소유한다. 아래는 로컬 개발에서만 적용하는 결정이며, 실제 설정 절차는 백엔드 저장소 README가 소유한다.

| 항목 | 결정 |
| --- | --- |
| 시크릿 주입 | 환경변수로 주입한다. 설정 파일에는 플레이스홀더만 둔다 |
| 메일 발송 | MailHog 또는 Mailpit 대역을 사용한다 |

```yaml
# application.yml — 값이 아니라 플레이스홀더만 둔다
jwt:
  secret: ${JWT_SECRET}
kakao:
  client-id: ${KAKAO_CLIENT_ID}
  client-secret: ${KAKAO_CLIENT_SECRET}
```

값을 적은 `application-local.yml`을 두는 방식은 사용하지 않는다. `.gitignore` 등록을 한 번 빠뜨리면 그대로 커밋되고, 형상관리 이력에서 제거하기 어렵다 (`NFR-CRED-007`).

메일 대역은 웹 UI로 수신 메일을 확인할 수 있어야 한다. 소유 확인·비밀번호 재설정은 코드가 메일 본문에만 존재하므로, 로그 출력만으로는 화면 흐름을 검증하기 어렵다.

## 후속 문서로 이관할 항목

**클라우드 아키텍처 문서**

- 컨테이너 오케스트레이션 구성
- 네트워크 구성과 접근 경로
- 컨테이너 레지스트리
- CI/CD 파이프라인 (GitHub Actions + Argo CD)
- Infrastructure as Code
- 인증정보 저장 및 주입 방식
- 데이터베이스와 Redis의 배치 위치

## 미결정 사항

| **ID** | **항목** | **후속 문서** |
| --- | --- | --- |
| ADR-016 | 소셜 제공자 추가 (Google 등) | 신규 ADR |
| ADR-016 | 운영 환경 메일 발송 수단 | 클라우드 아키텍처 |
| ADR-017 | 계정 연결 기능 (한 계정에 인증 수단 추가) | 신규 ADR |
| ADR-018 | 제재 회피 방지 (탈퇴 후 재가입 제한) | 신규 ADR |

확정된 항목: WebSocket은 연결 후 `authenticate` 이벤트로 토큰을 전달한다. 1차 릴리스 썸네일은 원본 서명 URL을 사용하고, 감사 기록은 PostgreSQL `audit_logs`에 조치와 같은 트랜잭션으로 저장한다.

API 서버와 채팅 서버 코드는 `service-backend` 한 저장소에 두며, 채팅 서버는 그 안의 `chat-server/` 별도 런타임으로 배포한다. `/api`는 Spring, `/socket.io`는 Socket.IO로 라우팅하며 WebSocket 업그레이드와 Origin 허용 목록이 필요하다.
