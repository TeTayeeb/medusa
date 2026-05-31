# File Module

## Overview

The `file` module is Medusa's stateless file storage abstraction layer. It exposes a uniform `IFileModuleService` interface for uploading, deleting, and generating presigned URLs for files, while delegating the actual storage operations to pluggable provider implementations (local disk or S3-compatible cloud storage).

## Purpose

Medusa features — product image upload, CSV import/export of products and prices, export of orders — all need to store and retrieve binary files. Rather than coupling these features to a specific storage backend, Medusa routes all file I/O through the file module's stable interface. Operators choose the provider that fits their infrastructure (local disk for development; AWS S3, Cloudflare R2, MinIO, etc. for production).

## Key Features

- **Provider abstraction** — the module itself contains no storage logic; it delegates to a registered `IFileProvider` implementation.
- **Upload** — accepts a `ReadableStream` plus metadata and returns the file's public URL and unique `key`.
- **Delete** — accepts a file `key` and removes it from the backing storage.
- **Presigned URLs** — generates time-limited, pre-authenticated URLs for direct browser downloads without exposing credentials.
- **Stateless** — the module holds no database entities; it is a pure service facade.
- **Multiple file types** — handles binary assets (images), text files (CSV), and any arbitrary MIME type.

## Interface

```typescript
interface IFileModuleService {
  upload(file: ProviderUploadFileDTO): Promise<ProviderFileResultDTO>
  delete(file: ProviderDeleteFileDTO): Promise<void>
  getPresignedDownloadUrl(file: ProviderGetFileDTO): Promise<string>
}
```

### DTO Types

```typescript
interface ProviderUploadFileDTO {
  filename: string
  mimeType: string
  content: string         // base64-encoded or the file's binary buffer
  access: "public" | "private"
}

interface ProviderFileResultDTO {
  url: string             // public access URL
  key: string             // unique storage key for later reference
}

interface ProviderDeleteFileDTO {
  fileKey: string
}

interface ProviderGetFileDTO {
  fileKey: string
  isPrivate?: boolean
}
```

## Admin API

The file module is exposed via:

```
POST   /admin/uploads           — Upload one or more files
DELETE /admin/uploads           — Delete a file by key
POST   /admin/uploads/protected — Upload a private (presigned-URL) file
GET    /admin/uploads/:key      — Get presigned download URL
```

## Configuration

```typescript
import { Modules } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/file",
      key: Modules.FILE,
      options: {
        // provider-specific options passed through to the active provider
        provider: "local",        // or "s3"
        upload_dir: "uploads",    // for local provider
      },
    },
  ],
})
```

## Built-in Providers

| Provider | Package | Use Case |
|---|---|---|
| Local disk | `@medusajs/file-local` | Development — writes files to `./uploads/` |
| AWS S3 / S3-compatible | `@medusajs/file-s3` | Production — AWS S3, MinIO, Cloudflare R2 |

## When to Use Which Provider

| Scenario | Recommendation |
|---|---|
| Local development | `file-local` (default) |
| Production — AWS | `file-s3` with S3 bucket |
| Production — self-hosted | `file-s3` pointed at MinIO |
| Testing | `file-local` or mock provider |

## Related Modules

- There are no direct module dependencies; the file module is consumed by product, order, and import/export workflows.
