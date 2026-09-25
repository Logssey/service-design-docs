# 프로필 이미지 연결 — schema/003_profile_images.sql

기존 `001_init.sql`은 수정하지 않는다. 운영·로컬 DB에 001, 002 다음으로 아래 SQL을 한 번 적용한 뒤 PROFILE 지원 API를 배포한다.

상품과 프로필 업로드는 공통 검증·소유권·정리 흐름을 사용한다. 기존 `listing_images` 테이블은 호환성을 위해 이름을 유지하며 `purpose`로 구분한다. 프로필은 한 회원당 하나만 연결하고 업로더 본인에게만 연결할 수 있다.

```sql
ALTER TABLE listing_images
    ADD COLUMN purpose VARCHAR(10) NOT NULL DEFAULT 'LISTING',
    ADD COLUMN profile_user_id BIGINT,
    ADD CONSTRAINT ck_images_purpose CHECK (purpose IN ('LISTING', 'PROFILE')),
    ADD CONSTRAINT fk_images_profile_user FOREIGN KEY (profile_user_id) REFERENCES users(user_id) ON DELETE RESTRICT,
    ADD CONSTRAINT ck_images_attachment CHECK (
        (purpose = 'LISTING' AND profile_user_id IS NULL) OR
        (purpose = 'PROFILE' AND listing_id IS NULL AND (profile_user_id IS NULL OR profile_user_id = uploader_id))
    );
CREATE UNIQUE INDEX uq_images_profile_user ON listing_images(profile_user_id) WHERE profile_user_id IS NOT NULL;
COMMENT ON COLUMN listing_images.purpose IS 'LISTING: 상품, PROFILE: 프로필. 기존 상품 이미지 기본값 유지';
COMMENT ON COLUMN listing_images.profile_user_id IS '현재 프로필 연결. 교체·해제·탈퇴 시 NULL로 해제 후 고아 정리';
```

## 저장·조회 규칙

- PROFILE 업로드는 `pending/profile-images/{uploaderId}/{uuid}` → 실제 바이트 검증 → `verified/profile-images/...`로 승격한다.
- `PATCH /users/me`의 `imageId`는 본인이 업로드한 VERIFIED·PROFILE 이미지여야 한다. 생략하면 유지하고 명시적 `null`은 기본 이미지로 되돌린다.
- 연결 변경과 `users.profile_image_url`의 내부 object key 변경은 하나의 트랜잭션이다. 외부 OAuth 아바타 URL은 기존 값으로 허용하되 API로 임의 URL을 입력받지 않는다.
- 응답에서는 내부 key를 노출하지 않고 매번 짧은 수명의 서명 URL로 변환한다. 만료되는 서명 URL 자체는 DB에 저장하지 않는다.
- 현재 연결된 이미지는 삭제 API·고아 정리 대상에서 제외한다. 교체·탈퇴 후 연결 해제된 이미지는 기존 보존 기간(업로드 후 24시간)·재시도 정리 정책을 따른다.
- PROFILE 이미지는 상품에 연결할 수 없고 LISTING 이미지는 프로필에 연결할 수 없다.
- 기존 S3 버킷·권한·Lifecycle·환경변수를 공유한다. 새 비밀키는 필요하지 않다.
