# SpecKit — File Module

**Module:** `@medusajs/file`
**Version:** 2.15.4
**Spec Status:** Approved

---

## 1. Module Identity

| Property | Value |
|---|---|
| Package name | `@medusajs/file` |
| DI key | `Modules.FILE` |
| Interface | `IFileModuleService` |
| Category | Infrastructure / File Storage |
| Default for | All Medusa installations (with `file-local` provider) |
| Admin API | `POST /admin/uploads`, `DELETE /admin/uploads` |

---

## 2. Functional Specifications

### SPEC-FIL-001 — Upload returns URL and key
**Given** a valid `ProviderUploadFileDTO` (filename, mimeType, content, access)
**When** `file.upload(dto)` is called
**Then** the result contains a non-empty `url` (publicly or privately accessible URL).
**And** the result contains a non-empty `key` (opaque storage reference).

### SPEC-FIL-002 — Upload delegates to provider
**Given** an `IFileProvider` mock is registered
**When** `file.upload(dto)` is called
**Then** `provider.upload(dto)` is invoked exactly once with the same DTO.
**And** the result from `provider.upload()` is returned unmodified.

### SPEC-FIL-003 — Delete removes file
**Given** a file has been uploaded and its `key` is known
**When** `file.delete({ fileKey: key })` is called
**Then** `provider.delete({ fileKey: key })` is invoked.
**And** subsequent access to the file's URL returns a 404 (provider-specific).

### SPEC-FIL-004 — Delete delegates to provider
**Given** an `IFileProvider` mock is registered
**When** `file.delete({ fileKey: "abc" })` is called
**Then** `provider.delete({ fileKey: "abc" })` is invoked exactly once.

### SPEC-FIL-005 — Presigned URL generated for private file
**Given** a private file was uploaded with `access: "private"`
**When** `file.getPresignedDownloadUrl({ fileKey: key, isPrivate: true })` is called
**Then** the result is a valid, time-limited URL string.
**And** the URL allows download of the file content without additional credentials (presigned by the provider).

### SPEC-FIL-006 — getPresignedDownloadUrl delegates to provider
**Given** an `IFileProvider` mock is registered
**When** `file.getPresignedDownloadUrl({ fileKey: "abc" })` is called
**Then** `provider.getPresignedDownloadUrl({ fileKey: "abc" })` is invoked exactly once.
**And** the returned string is the provider's result.

### SPEC-FIL-007 — Admin API upload endpoint
**Given** a valid `multipart/form-data` request to `POST /admin/uploads`
**When** it reaches the API route handler
**Then** `file.upload()` is called for each uploaded file.
**And** the response status is `200` with body `{ files: [{ url, key }] }`.

### SPEC-FIL-008 — Admin API delete endpoint
**Given** a `DELETE /admin/uploads` request with body `{ file_key: "abc" }`
**When** it reaches the API route handler
**Then** `file.delete({ fileKey: "abc" })` is called.
**And** the response status is `200` with body `{ id: "abc", object: "file", deleted: true }`.

### SPEC-FIL-009 — No stateful side effects
**Given** any number of `upload()` and `delete()` calls
**Then** the `FileModuleService` itself has no internal state (no maps, no caches, no counters).
**And** the service can be disposed and re-instantiated with identical results.

---

## 3. Non-Functional Specifications

### SPEC-FIL-NF-001 — Stateless module
`FileModuleService` must have no MikroORM entities and no database dependencies.

### SPEC-FIL-NF-002 — Provider swappability
Changing the active provider (local ↔ S3) must require only a configuration change, not a code change.

### SPEC-FIL-NF-003 — Custom provider support
A custom `IFileProvider` implementation must be injectable via the DI container without modifying core module code.

---

## 4. Configuration Specification

```typescript
interface FileModuleOptions {
  provider?: string       // "local" | "s3" | custom provider key (default: "local")
  // Provider-specific options passed through to the provider constructor
  [key: string]: unknown
}
```

---

## 5. Acceptance Criteria Matrix

| Spec ID | Test Type | Status |
|---|---|---|
| SPEC-FIL-001 | Integration (local provider) | Required |
| SPEC-FIL-002 | Unit (mock provider) | Required |
| SPEC-FIL-003 | Integration (local provider) | Required |
| SPEC-FIL-004 | Unit (mock provider) | Required |
| SPEC-FIL-005 | Integration (S3 provider) | Optional |
| SPEC-FIL-006 | Unit (mock provider) | Required |
| SPEC-FIL-007 | HTTP integration | Required |
| SPEC-FIL-008 | HTTP integration | Required |
| SPEC-FIL-009 | Unit | Required |
| SPEC-FIL-NF-001 | Static analysis (no entities) | Required |
| SPEC-FIL-NF-002 | Integration (swap providers) | Required |
| SPEC-FIL-NF-003 | Unit (custom mock provider) | Required |

---

## 6. Out of Scope

- Image resizing or format conversion
- CDN cache invalidation
- File type allow-listing (enforced at API route layer)
- File metadata storage (stored by consuming modules)
- Access-control policy management
- Virus/malware scanning
