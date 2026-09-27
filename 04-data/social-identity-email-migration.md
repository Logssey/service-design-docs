# 소셜 인증 수단 이메일 — schema/004_social_identity_email.sql

기존 `001_init.sql`과 `003_profile_images.sql`은 수정하지 않는다. 운영·로컬 DB에 003 다음으로 아래 SQL을 한 번 적용한 뒤 소셜 선택 이메일을 지원하는 애플리케이션을 배포한다 (ADR-016).

소셜 온보딩에서 사용자가 선택 입력한 이메일과 그 수집·이용 동의 시각을 소셜 인증 수단에 허용한다. 소유 확인이 끝난 소셜 이메일은 소셜 인증 수단 전체에서 하나로 제한한다. 비밀번호는 계속 `LOCAL` 전용이다.

## 적용 시점

반드시 애플리케이션 배포 **전에** 적용한다.

- 기존 소셜 행은 이메일과 소유 확인 시각이 모두 `NULL`이라 이관 없이 새 제약을 만족한다. 기존 버전의 애플리케이션이 쓰는 값도 모두 만족하므로, 먼저 적용해도 기존 버전과 호환된다.
- 새 애플리케이션은 `email_consent_at` 컬럼을 읽고 쓴다. 004를 적용하지 않은 DB에서는 인증 수단을 조회·저장하는 모든 요청(소셜·이메일 로그인, 이메일 가입, 소셜 온보딩, 소유 확인 코드 발송·확인, 비밀번호 재설정·변경)이 이메일 입력 여부와 무관하게 `500`으로 실패한다.

적용 전에 아래 결과가 0인지 확인한다.

```bash
psql -U postgres -d reused -c "SELECT count(*) FROM user_identities WHERE email IS NULL AND email_verified_at IS NOT NULL"
```

## SQL

```sql
-- ============================================================
-- Re:Used 스키마 변경
-- 파일: schema/004_social_identity_email.sql
-- 소셜 인증 수단에 온보딩에서 사용자가 선택 입력한 이메일과 그 수집·이용 동의 시각을 허용한다 (ADR-016)
-- 비밀번호는 계속 LOCAL 전용이다
-- 소유 확인이 끝난 소셜 이메일은 소셜 인증 수단 전체에서 하나만 둔다. 확인 전 주소와 LOCAL 이메일은 겹쳐도 된다
-- uq_user_identities_email_local 과 ck_user_identities_local_email 은 그대로 둔다
-- 기존 소셜 행은 이메일이 모두 NULL이라 이관 없이 새 제약을 만족한다
-- 제약 완화와 NULL 허용 컬럼 추가라 애플리케이션 배포 전에 적용해도 기존 버전과 호환된다
-- ============================================================

ALTER TABLE user_identities
    DROP CONSTRAINT ck_user_identities_social_credential,
    ADD CONSTRAINT ck_user_identities_social_credential CHECK (
        provider = 'LOCAL' OR password_hash IS NULL),
    ADD CONSTRAINT ck_user_identities_verified_email CHECK (
        email_verified_at IS NULL OR email IS NOT NULL),
    ADD COLUMN email_consent_at TIMESTAMPTZ,
    ADD CONSTRAINT ck_user_identities_social_email_consent CHECK (
        provider = 'LOCAL' OR email IS NULL OR email_consent_at IS NOT NULL);

-- 소유 확인이 끝난 소셜 이메일은 한 계정에만 둔다. 동시에 확인해도 한 건만 통과한다.
CREATE UNIQUE INDEX uq_user_identities_email_social_verified
    ON user_identities (email) WHERE provider <> 'LOCAL' AND email_verified_at IS NOT NULL;

COMMENT ON COLUMN user_identities.email IS 'LOCAL 필수, LOCAL 범위에서만 유일. 소셜은 온보딩 선택 입력 연락처이며 소유 확인이 끝난 주소만 소셜 범위에서 유일 (ADR-016)';
COMMENT ON COLUMN user_identities.password_hash IS 'LOCAL 필수, 소셜은 항상 NULL. 고유 Salt 적응형 단방향 해시';
COMMENT ON COLUMN user_identities.email_verified_at IS '이메일 소유 확인 시각. NULL이면 미인증. 이메일이 없으면 항상 NULL';
COMMENT ON COLUMN user_identities.email_consent_at IS '소셜 이메일 수집·이용 선택 동의 시각. 소셜 인증 수단에 이메일이 있으면 필수. LOCAL은 쓰지 않음 (ADR-016)';
```

## 규칙

- 입력 단계(소셜 온보딩)에서는 이메일 중복을 검사하지 않는다. 확인 전 주소는 선점할 수 없고, 가입 여부도 드러나지 않는다.
- 소유 확인 시 같은 주소가 다른 소셜 인증 수단에서 이미 확인되어 있으면 `409`다. 동시에 확인해도 부분 UNIQUE 인덱스 `uq_user_identities_email_social_verified`가 한 건만 통과시킨다.
- `LOCAL` 이메일과는 겹쳐도 된다. 제공자가 다르면 별개 계정이며 병합하지 않는다 (ADR-016). `LOCAL` 범위 유일성(`uq_user_identities_email_local`)은 그대로다.
- 소셜 이메일이 있으면 수집·이용 동의 시각(`email_consent_at`)도 있어야 한다. `LOCAL`은 이 컬럼을 쓰지 않는다.
- 이메일 로그인·가입 중복 검사·비밀번호 재설정 대상 조회는 반드시 `provider = 'LOCAL'` 조건을 함께 건다 ([스키마 및 ERD](schema-and-erd.md) 애플리케이션 레벨 주의사항 11).

## 적용 후 확인

```bash
psql -U postgres -d reused -c "\d user_identities"
```
