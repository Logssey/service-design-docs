# 이미지 업로드 URL 발급

도메인: IMAGES
Method: POST
URL: /api/v1/images/upload-url
Version: 1
개발완료여부: Yes

Presigned URL 발급 (ADR-011).

**인증** `USER`

**Request** `ImageUploadUrlRequest`

**Response** `ImageUploadUrlResponse`

```java
// ImageUploadUrlRequest
{
  "purpose": "LISTING",           // @NotNull — LISTING | PROFILE
  "fileName": "ipad.jpg",         // @NotBlank
  "contentType": "image/jpeg",    // @NotNull — image/jpeg | image/png | image/webp
  "fileSize": 2048000             // @NotNull @Max(10485760)
}

// ImageUploadUrlResponse (201)
{
  "imageId": 7,
  "uploadUrl": "https://s3.../presigned?...",
  "expiresAt": "2026-03-15T09:35:00Z"
}
```

| 오류 | 상황 |
| --- | --- |
| 400 `INVALID_INPUT` | 허용되지 않는 형식, 10MB 초과 |

관련: `FR-IMG-001`, `FR-IMG-002`
