# @medusajs/file-s3

AWS S3 file storage provider for Medusa v2. Implements `IFileProvider` via `AbstractFileProviderService` and registers under `Modules.FILE` with identifier `s3`.

Supports standard file operations (upload, delete, presigned download/upload URLs) and streaming I/O using **AWS SDK v3** (`@aws-sdk/client-s3`).

## Installation

```bash
npm install @medusajs/file-s3
```

## Configuration

```ts
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/medusa/file",
      options: {
        providers: [
          {
            resolve: "@medusajs/file-s3",
            id: "s3",
            options: {
              file_url: process.env.S3_FILE_URL,       // Public base URL for file access
              region: process.env.S3_REGION,           // e.g. "us-east-1"
              bucket: process.env.S3_BUCKET,           // S3 bucket name
              access_key_id: process.env.S3_ACCESS_KEY_ID,
              secret_access_key: process.env.S3_SECRET_ACCESS_KEY,
            },
          },
        ],
      },
    },
  ],
})
```

### Full options reference

| Option | Type | Required | Default | Description |
|---|---|---|---|---|
| `file_url` | `string` | ✅ | — | Public base URL (e.g. `https://bucket.s3.amazonaws.com`) |
| `region` | `string` | ✅ | — | AWS region |
| `bucket` | `string` | ✅ | — | S3 bucket name |
| `access_key_id` | `string` | ✅* | — | AWS access key ID. *Required when `authentication_method` is `"access-key"` (default) |
| `secret_access_key` | `string` | ✅* | — | AWS secret access key |
| `authentication_method` | `"access-key"` \| `"iam"` | — | `"access-key"` | `"iam"` uses the AWS default credential chain (EC2 role, ECS task role, etc.) |
| `session_token` | `string` | — | — | STS session token for temporary credentials |
| `prefix` | `string` | — | `""` | Key prefix prepended to all uploaded files |
| `endpoint` | `string` | — | — | Custom endpoint (for S3-compatible stores like MinIO, Cloudflare R2) |
| `cache_control` | `string` | — | `"public, max-age=31536000"` | `Cache-Control` header on uploaded objects |
| `download_file_duration` | `number` | — | `3600` | Presigned download URL expiry in seconds |
| `additional_client_config` | `object` | — | `{}` | Passed directly to `S3Client` constructor |

## Key operations

### `upload(file)`
- Generates a unique key: `<prefix><name>-<ULID><ext>`
- Detects base64 content; falls back to UTF-8 then binary
- Calls `PutObjectCommand` with `ACL: "public-read"` for public files, `"private"` for private
- Returns `{ url: "<file_url>/<encoded-key>", key }`

### `delete(files)`
- Single file: `DeleteObjectCommand`
- Array of files: `DeleteObjectsCommand` (bulk delete)
- Errors are logged but not re-thrown (best-effort deletion)

### `getPresignedDownloadUrl(fileData)`
- Calls `getSignedUrl` with `GetObjectCommand`
- Expiry: `download_file_duration` seconds (default 1 hour)

### `getPresignedUploadUrl(fileData)`
- Returns a presigned `PUT` URL for direct client-to-S3 uploads
- Default expiry: 1 hour (`DEFAULT_UPLOAD_EXPIRATION_DURATION_SECONDS = 3600`)

### `getUploadStream(fileData)` / `getDownloadStream(file)` / `getAsBuffer(file)`
Advanced streaming APIs using `@aws-sdk/lib-storage` `Upload` class for multipart upload support.

## S3-compatible providers (MinIO, R2, etc.)

Set `endpoint` and adjust `authentication_method` as needed:
```ts
options: {
  endpoint: "https://your-minio.example.com",
  region: "us-east-1",           // required even for non-AWS
  authentication_method: "access-key",
  access_key_id: "minioadmin",
  secret_access_key: "minioadmin",
  bucket: "medusa",
  file_url: "https://your-minio.example.com/medusa",
}
```

## Environment variables

```dotenv
S3_REGION=us-east-1
S3_BUCKET=my-medusa-bucket
S3_ACCESS_KEY_ID=AKIA...
S3_SECRET_ACCESS_KEY=...
S3_FILE_URL=https://my-medusa-bucket.s3.us-east-1.amazonaws.com
```
