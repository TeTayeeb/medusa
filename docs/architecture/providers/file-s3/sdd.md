# Software Design Document — @medusajs/file-s3

## 1. Purpose

Provide production-grade file storage on AWS S3 (and S3-compatible services) for Medusa's File Module. Handles upload, deletion, presigned URL generation, and streaming I/O with support for multiple authentication methods and S3-compatible endpoints.

## 2. Architecture

```
Modules.FILE
  └── ModuleProvider (file-s3)
        └── S3FileService (AbstractFileProviderService)
              ├── upload(file)                      → { url, key }
              ├── delete(files)                     → void
              ├── getPresignedDownloadUrl(fileData) → string (URL)
              ├── getPresignedUploadUrl(fileData)   → { url, key }
              ├── getUploadStream(fileData)         → { writeStream, promise, url, fileKey }
              ├── getDownloadStream(file)           → ReadableStream
              ├── getAsBuffer(file)                 → Buffer
              └── getClient()                       → S3Client
```

All S3 operations use AWS SDK v3 (`@aws-sdk/client-s3`) for tree-shaking and modular imports.

## 3. File key generation

Every uploaded file receives a unique key:
```
key = `${prefix}${name}-${ulid()}${ext}`
```

- `ulid()` provides time-ordered, URL-safe unique IDs (no UUID collision risk).
- The ULID is appended before the extension to preserve the original filename semantics.
- The returned URL percent-encodes each path segment: `key.split("/").map(encodeURIComponent).join("/")`.

## 4. Authentication methods

### `"access-key"` (default)
```ts
credentials: {
  accessKeyId: config.accessKeyId,
  secretAccessKey: config.secretAccessKey,
  sessionToken: config.sessionToken,  // optional STS
}
```
Validated at construction time — both `access_key_id` and `secret_access_key` must be present.

### `"iam"`
`credentials` is set to `undefined`, which triggers the AWS SDK's default credential provider chain (environment variables → EC2 instance metadata → ECS task role → etc.). Suitable for deployments on AWS infrastructure.

## 5. Content encoding

Uploaded file content arrives as a string. The provider attempts base64 decoding first:
```ts
const decoded = Buffer.from(file.content, "base64")
if (decoded.toString("base64") === file.content) {
  content = decoded  // was valid base64
} else {
  content = Buffer.from(file.content, "utf8")  // plain text
}
// fallback: Buffer.from(file.content, "binary")
```

## 6. Access control

`ACL` is set per-file:
- `file.access === "public"` → `ACL: "public-read"`
- otherwise → `ACL: "private"`

> **Note**: Bucket-level ACL policies or Block Public Access settings can override per-object ACLs. Ensure the bucket policy aligns with the access model.

## 7. Presigned URLs

### Download
`GetObjectCommand` signed with `getSignedUrl` from `@aws-sdk/s3-request-presigner`. Expiry: configurable `download_file_duration` (default 3600 s).

### Upload (client-side direct upload)
`PutObjectCommand` signed URL allowing the client to upload directly to S3, bypassing the Medusa server. Expiry: `fileData.expiresIn` or `DEFAULT_UPLOAD_EXPIRATION_DURATION_SECONDS` (3600 s).

## 8. Streaming upload

Large file uploads use `@aws-sdk/lib-storage` `Upload` class, which:
- Automatically uses multipart upload for files > 5 MB
- Wraps a `PassThrough` stream for piping
- Returns a `Promise<{ url, key }>` that resolves when the S3 multipart upload completes

## 9. Bulk delete

```ts
delete(files: FileDTO[]):
  if Array.isArray(files):
    DeleteObjectsCommand({ Delete: { Objects: files.map(f => ({ Key: f.fileKey })) } })
  else:
    DeleteObjectCommand({ Key: files.fileKey })
```

Errors are caught and logged (not rethrown) since file-not-found scenarios are non-fatal in most workflows.

## 10. S3-compatible endpoints

The `endpoint` config option is passed directly to `S3Client`. Combined with `forcePathStyle` in `additional_client_config`, this supports MinIO, Cloudflare R2, DigitalOcean Spaces, and other S3-compatible services.

## 11. Cache control

All uploads include `CacheControl: config.cacheControl` (default `"public, max-age=31536000"` — 1 year). This is appropriate for media assets that use content-addressed keys (ULID-based). Adjust for frequently-updated assets.

## 12. Dependencies

| Package | Purpose |
|---|---|
| `@aws-sdk/client-s3` | Core S3 commands |
| `@aws-sdk/lib-storage` | Multipart streaming upload |
| `@aws-sdk/s3-request-presigner` | Presigned URL generation |
| `ulid` | Unique file key generation |
| `path` | File extension/name parsing |
| `@medusajs/framework` | `AbstractFileProviderService`, `MedusaError` |
