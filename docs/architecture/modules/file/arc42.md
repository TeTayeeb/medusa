# arc42 Architecture Documentation — File Module

**Module:** `@medusajs/file`
**Version:** 2.15.4

---

## 1. Introduction and Goals

### 1.1 Requirements Overview
The file module must provide a stable, provider-agnostic file storage interface to all Medusa features. It must support upload, delete, and presigned-URL generation, and delegate the actual storage to a pluggable `IFileProvider`. It is stateless — it maintains no database entities.

### 1.2 Quality Goals

| Priority | Quality Goal | Scenario |
|---|---|---|
| 1 | **Provider agnosticism** | Swapping from local disk to S3 requires only config change |
| 2 | **Simplicity** | The service is a thin facade; no business logic |
| 3 | **Extensibility** | Custom providers can be plugged in without modifying core code |

---

## 2. Architecture Constraints

- `FileModuleService` must not contain any storage-specific code.
- Must expose Admin API at `/admin/uploads` for all file operations.
- Provider selection must be config-driven (no code change to swap providers).
- The module is stateless — no MikroORM entities.

---

## 3. System Scope and Context

```
┌─────────────────────────────────────────────────────────────────┐
│                      Medusa Application                         │
│                                                                 │
│  ProductModule: upload image ──► file.upload()                  │
│  ImportWorkflow: upload CSV  ──► file.upload()                  │
│  ExportWorkflow: download    ──► file.getPresignedDownloadUrl() │
│                                                                 │
│         ┌───────────────────────────────────────────┐          │
│         │   FileModuleService (facade)               │          │
│         └───────────────────────┬─────────────────┘          │
│                                 │                               │
│               ┌─────────────────┴─────────────────┐           │
│               ▼                                   ▼            │
│         LocalFileProvider                  S3FileProvider       │
│         (./uploads/ on disk)              (AWS S3 / R2 / MinIO) │
└─────────────────────────────────────────────────────────────────┘
```

**External interfaces:**
- **Local provider** — filesystem (`fs.writeFile`, `fs.unlink`)
- **S3 provider** — AWS SDK v3 (`PutObject`, `DeleteObject`, `GetObjectCommand` presign)

---

## 4. Solution Strategy

`FileModuleService` is a strict delegation facade: it accepts an `IFileProvider` at construction time (injected by Medusa's DI container) and routes all three operations directly to the provider. No caching, no retry, no transformation. The provider encapsulates all storage-specific logic. Providers are resolved from the DI container based on the `provider` configuration key.

---

## 5. Building Blocks

```
file
  ├── FileModuleService        (IFileModuleService impl — facade only)
  ├── IFileProvider            (interface contract for providers)
  ├── LocalFileProvider        (dev: writes to ./uploads/)
  └── S3FileProvider           (prod: AWS SDK v3)
```

### Delegation Contract

```
FileModuleService.upload(dto)
  └──► IFileProvider.upload(dto)
         └──► { url, key }

FileModuleService.delete(dto)
  └──► IFileProvider.delete(dto)

FileModuleService.getPresignedDownloadUrl(dto)
  └──► IFileProvider.getPresignedDownloadUrl(dto)
         └──► presigned URL string
```

---

## 6. Runtime View

### Scenario: Product Image Upload

```
Admin UI: POST /admin/uploads (multipart/form-data)
  │
  ▼ API Route Handler
  parseMultipart(req)
    │── { filename: "hero.jpg", mimeType: "image/jpeg", content: <binary> }
    │
    ▼ FileModuleService.upload(dto)
      │
      ▼ S3FileProvider.upload(dto)
        ├── AWS SDK: PutObject(Bucket, Key=<uuid>/hero.jpg, Body=<binary>)
        └── return { url: "https://cdn.example.com/hero.jpg", key: "abc123/hero.jpg" }
  │
  ▼ Response: { files: [{ url, key }] }
```

---

## 7. Deployment View

```
┌──────── Production ─────────────────────────────────────────────┐
│                                                                  │
│  Medusa API Pod          FileModuleService (stateless)           │
│  POST /admin/uploads ──► S3FileProvider ──► AWS S3 / MinIO       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

Local development:

```
Node.js Process
  FileModuleService → LocalFileProvider → ./uploads/ (host filesystem)
```

---

## 8. Cross-Cutting Concepts

### Statelessness
The file module has no MikroORM entities. File metadata (URL, key) is stored by the consuming module (e.g., `product_image.url`). The file module only manages the binary blob.

### Security
- Private files use presigned URLs with expiry (provider-managed).
- File type validation (MIME type allow-listing) is the responsibility of the API route, not the module.

---

## 9. Architecture Decisions

| ID | Decision | Rationale |
|---|---|---|
| AD-1 | Stateless facade pattern | Single responsibility; module owns only the I/O contract |
| AD-2 | IFileProvider interface | Enables custom providers without modifying core code |
| AD-3 | No caching of presigned URLs | TTL management is complex and provider-specific |

---

## 10. Quality Requirements

| Requirement | Measure |
|---|---|
| Provider swappability | Integration test: same test suite passes against both local and S3 providers |
| Upload correctness | Integration test: uploaded file readable at returned URL |
| Delete correctness | Integration test: file absent after delete |

---

## 11. Risks and Technical Debt

| Risk | Impact | Mitigation |
|---|---|---|
| Storage backend unavailability | High | S3 SLA covers this; local disk has no HA |
| Large file upload memory pressure | Medium | Stream uploads rather than buffering entire file in memory |
| No file type validation in module | Low-Medium | Implement in API route layer; document this responsibility |
