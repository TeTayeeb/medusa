# Software Design Document — File Module

**Module:** `@medusajs/file`
**Version:** 2.15.4
**Status:** Stable

---

## 1. Overview

The `file` module is a stateless provider-abstraction service that delegates all file storage operations to a configured `IFileProvider` implementation. It implements `IFileModuleService` and is consumed by Medusa features that need to store or retrieve binary assets: product images, CSV imports/exports, and order exports.

---

## 2. Goals and Non-Goals

### Goals
- Provide a stable, provider-agnostic file storage interface to the rest of Medusa.
- Support `upload`, `delete`, and `getPresignedDownloadUrl` operations.
- Allow operators to choose between local-disk and S3-compatible storage via configuration.
- Expose operations via the Admin API at `/admin/uploads`.

### Non-Goals
- Image resizing or transformation.
- CDN management.
- Access-control policies beyond what the provider natively supports.
- Tracking file metadata in a database.

---

## 3. Architecture

### 3.1 Component Model

```
Medusa Feature (e.g., product image upload)
          │
          ▼
┌─────────────────────────────────────────────┐
│        FileModuleService                    │
│  ─────────────────────────────────────────  │
│  + upload(file): Promise<FileResult>        │
│  + delete(file): Promise<void>              │
│  + getPresignedDownloadUrl(file): Promise<string> │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │  IFileProvider (injected)           │   │
│  │  (resolved from DI container)       │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
         │                   │
         ▼                   ▼
  LocalFileProvider    S3FileProvider
  (disk write)         (AWS SDK PutObject)
```

### 3.2 Key Classes

| Class / File | Responsibility |
|---|---|
| `FileModuleService` | Thin facade; resolves the active provider, delegates all calls |
| `IFileProvider` | Interface that all provider implementations must satisfy |
| `LocalFileProvider` | Writes files to `./uploads/`; returns relative URLs |
| `S3FileProvider` | Uses AWS SDK v3 to upload to S3; generates pre-signed URLs via `GetObjectCommand` |
| Module definition | Registers `FileModuleService` and the selected provider |
| API route handler | `POST /admin/uploads` — parses multipart form-data, calls `upload()` |

### 3.3 Provider Interface

```typescript
interface IFileProvider {
  upload(file: ProviderUploadFileDTO): Promise<ProviderFileResultDTO>
  delete(file: ProviderDeleteFileDTO): Promise<void>
  getPresignedDownloadUrl(file: ProviderGetFileDTO): Promise<string>
}
```

`FileModuleService` is a strict pass-through:

```typescript
class FileModuleService implements IFileModuleService {
  constructor(private readonly provider: IFileProvider) {}

  async upload(file: ProviderUploadFileDTO) {
    return this.provider.upload(file)
  }

  async delete(file: ProviderDeleteFileDTO) {
    return this.provider.delete(file)
  }

  async getPresignedDownloadUrl(file: ProviderGetFileDTO) {
    return this.provider.getPresignedDownloadUrl(file)
  }
}
```

---

## 4. Admin API Design

### `POST /admin/uploads`
- Accepts `multipart/form-data` with one or more file fields.
- Each file is converted to `ProviderUploadFileDTO` and passed to `upload()`.
- Response: `{ files: [{ url, key }] }`.

### `DELETE /admin/uploads`
- Body: `{ file_key: string }`.
- Calls `delete({ fileKey })`.
- Response: `{ id, object: "file", deleted: true }`.

### `POST /admin/uploads/protected`
- Same as `POST /admin/uploads` but sets `access: "private"`.
- Provider generates a presigned URL for browser access.

---

## 5. Data Structures

```typescript
interface ProviderUploadFileDTO {
  filename: string
  mimeType: string
  content: string        // base64-encoded binary or UTF-8 text
  access: "public" | "private"
}

interface ProviderFileResultDTO {
  url: string            // publicly accessible URL
  key: string            // opaque storage key for delete/presign
}

interface ProviderDeleteFileDTO {
  fileKey: string
}

interface ProviderGetFileDTO {
  fileKey: string
  isPrivate?: boolean
}
```

---

## 6. Error Handling

- Provider errors are propagated as-is to the caller.
- `MedusaError.Types.INVALID_DATA` is thrown if `filename` or `mimeType` is missing.
- HTTP 500 is returned if the provider fails to upload (e.g., S3 credentials invalid).

---

## 7. Dependencies

| Dependency | Purpose |
|---|---|
| `@medusajs/framework/types` | `IFileModuleService`, `IFileProvider` interfaces |
| `@medusajs/framework/utils` | Module registration |
| `@aws-sdk/client-s3` (optional) | Used by `file-s3` provider |
| `busboy` / multipart parser | Parses file uploads in the API route |

---

## 8. Testing Strategy

- Unit tests mock `IFileProvider` and verify `FileModuleService` correctly delegates all three operations.
- Integration tests use `file-local` provider: upload a file, verify it exists on disk, delete it, verify it is gone, request presigned URL.
- API-level integration tests exercise `POST /admin/uploads` with real multipart payloads.
