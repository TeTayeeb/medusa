# SpecKit — @medusajs/file-s3

---

## 1. Unit specs — `upload`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U1 | Happy path | `{ filename: "img.jpg", content: <base64>, mimeType: "image/jpeg", access: "public" }` | Returns `{ url: "<file_url>/img-<ULID>.jpg", key: "img-<ULID>.jpg" }` |
| U2 | Private file | `{ access: "private", ... }` | `PutObjectCommand` called with `ACL: "private"` |
| U3 | Public file | `{ access: "public", ... }` | `PutObjectCommand` called with `ACL: "public-read"` |
| U4 | Base64 content detected | Base64-encoded string | Content decoded correctly to binary buffer |
| U5 | Plain text content | UTF-8 string | Content stored as UTF-8 buffer |
| U6 | Prefix applied | Config `prefix: "uploads/"` | Key starts with `"uploads/"` |
| U7 | ULID in key | Any upload | Key matches `<prefix><name>-<ULID><ext>` pattern |
| U8 | URL percent-encoding | Filename with spaces | URL segments are percent-encoded |
| U9 | No file provided | `null` | Throws `MedusaError(INVALID_DATA, "No file provided")` |
| U10 | No filename | `{ content: "..." }` | Throws `MedusaError(INVALID_DATA, "No filename provided")` |
| U11 | Original filename in metadata | Any upload | S3 `Metadata["original-filename"]` set to `encodeURIComponent(filename)` |

---

## 2. Unit specs — `delete`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U12 | Single file delete | `{ fileKey: "img-xyz.jpg" }` | `DeleteObjectCommand` called |
| U13 | Bulk delete | Array of 3 file objects | `DeleteObjectsCommand` called with all 3 keys |
| U14 | S3 error on delete | S3 throws | Error is logged; no exception rethrown |

---

## 3. Unit specs — `getPresignedDownloadUrl`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U15 | Happy path | `{ fileKey: "img.jpg" }` | Returns a signed URL string |
| U16 | Custom expiry | `download_file_duration: 7200` | `expiresIn: 7200` passed to `getSignedUrl` |
| U17 | Default expiry | No config override | `expiresIn: 3600` |

---

## 4. Unit specs — `getPresignedUploadUrl`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U18 | Happy path | `{ filename: "doc.pdf", mimeType: "application/pdf" }` | Returns `{ url: <signed PUT URL>, key: "<prefix>doc.pdf" }` |
| U19 | No filename | `{ mimeType: "image/jpeg" }` | Throws `MedusaError(INVALID_DATA, "No filename provided")` |
| U20 | Custom expiry via `expiresIn` | `{ expiresIn: 300 }` | `getSignedUrl` called with `expiresIn: 300` |

---

## 5. Unit specs — client construction

| # | Scenario | Expected outcome |
|---|---|---|
| U21 | `authentication_method: "access-key"` without keys | Throws `MedusaError(INVALID_DATA)` at construction |
| U22 | `authentication_method: "iam"` | `S3Client` created with `credentials: undefined` |
| U23 | `endpoint` provided | `S3Client` created with custom endpoint |
| U24 | `additional_client_config` merged | Extra config forwarded to `S3Client` |

---

## 6. Integration specs

| # | Scenario | Expected outcome |
|---|---|---|
| I1 | Upload + download via presigned URL | File accessible at presigned URL |
| I2 | Upload + delete | File no longer accessible after delete |
| I3 | S3-compatible endpoint (MinIO) | Upload/download work with custom endpoint |
| I4 | IAM authentication on EC2 | No explicit credentials needed; uses instance role |

---

## 7. Acceptance criteria

- ULID keys ensure no two uploads share the same key even with identical filenames.
- Base64 and UTF-8 content are both handled correctly without corruption.
- Bulk delete processes ≥100 files in a single `DeleteObjectsCommand`.
- Presigned download URLs expire within the configured `download_file_duration`.
- Configuration validation errors are thrown synchronously at construction (fail-fast).
