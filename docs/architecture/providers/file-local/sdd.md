# Software Design Document — @medusajs/file-local

## 1. Purpose

Provide a zero-dependency file storage backend for local development. Files are written to the host filesystem and served via Medusa's Express static middleware. No external services are required, making it ideal for local development and CI environments.

## 2. Architecture

```
Modules.FILE
  └── ModuleProvider (file-local)
        └── LocalFileService (AbstractFileProviderService)
              ├── upload(file)                      → { url, key }
              ├── delete(files)                     → void
              ├── getPresignedDownloadUrl(fileData) → string (direct static URL)
              ├── getPresignedUploadUrl(fileData)   → { url: "/admin/uploads", key }
              ├── getUploadStream(fileData)         → { writeStream, promise, url, fileKey }
              ├── getDownloadStream(file)           → ReadStream
              ├── getAsBuffer(file)                 → Buffer
              └── ensureDirExists(baseDir, dirPath) → void
```

No external dependencies beyond Node.js built-ins (`fs`, `path`).

## 3. Directory structure

```
<process.cwd()>/
└── static/                   ← default upload_dir / private_upload_dir
    ├── 1715000000000-logo.png        (public file)
    ├── private-1715000001000-doc.pdf (private file, same dir)
    └── uploads/
        └── ...
```

Both public and private files default to the same `static/` directory. The `private-` prefix is a naming convention, not a security boundary.

## 4. File key construction

```ts
const fileKey = path.join(
  parsedFilename.dir,
  `${access === "public" ? "" : "private-"}${Date.now()}-${parsedFilename.base}`
)
```

Examples:
- Public: `1715000000000-product.jpg`
- Private: `private-1715000001000-invoice.pdf`
- In subdirectory: `receipts/private-1715000001000-receipt.pdf`

`Date.now()` provides rough uniqueness. Collisions are theoretically possible under high concurrency — acceptable for development, unacceptable for production.

## 5. Upload flow

```
upload(file):
  1. Validate file and filename are present.
  2. Parse filename (dir, name, ext).
  3. Determine baseDir (upload_dir for public, privateUploadDir for private).
  4. ensureDirExists(baseDir, parsedFilename.dir)
     → fs.mkdir({ recursive: true }) if not present
  5. Construct fileKey and filePath.
  6. Decode content: base64 → Buffer | UTF-8 Buffer | binary Buffer
  7. fs.promises.writeFile(filePath, content)
  8. return { key: fileKey, url: backendUrl + fileKey }
```

## 6. Delete flow

```
delete(files):
  files = Array.isArray(files) ? files : [files]
  for each file:
    baseDir = fileKey.startsWith("private-") ? privateUploadDir : uploadDir
    filePath = path.join(baseDir, fileKey)
    fs.promises.access(filePath, W_OK)
    fs.promises.unlink(filePath)
    catch ENOENT → silent (already deleted)
    catch other  → rethrow
```

## 7. Presigned URL behaviour

### `getPresignedDownloadUrl`
No time-limited URL is generated. Instead, the provider returns a direct static URL:
```ts
return this.getUploadFileUrl(file.fileKey)
// → "http://localhost:9000/static/<fileKey>"
```
It first verifies the file exists (`fs.promises.access(filePath, F_OK)`), throwing `MedusaError(NOT_FOUND)` if absent.

### `getPresignedUploadUrl`
Returns `{ url: "/admin/uploads", key: filename }`. This instructs the caller to submit the file to the Medusa server's upload endpoint — there is no client-to-disk direct upload path.

## 8. Content encoding

Same three-pass strategy as file-s3:
1. Try base64 decode; if `decoded.toString("base64") === input`, use decoded buffer.
2. Fall back to UTF-8 buffer.
3. Last resort: binary buffer.

## 9. URL construction

```ts
getUploadFileUrl(fileKey):
  baseUrl = new URL(this.backendUrl_)           // e.g. "http://localhost:9000/static"
  baseUrl.pathname = path.join(baseUrl.pathname, fileKey)
  return baseUrl.href
```

Using `URL` ensures correct path joining and encoding.

## 10. Streaming

`getUploadStream` wraps `fs.createWriteStream`. The returned `promise` resolves/rejects based on the stream's `finish`/`error` events. This allows the File Module to await completion without buffering the entire file in memory.

## 11. Private file limitations

The comment in the source is explicit:
> "Since there is no way to serve private files through a static server, we simply place them in `static`. This means that the files will be available publicly if the filename is known."

If `private_upload_dir` is overridden to a non-static path, presigned download URLs will still point to `backend_url + fileKey` and will not be accessible — use file-s3 for any private file use case in production.

## 12. Dependencies

| Package | Purpose |
|---|---|
| `fs` / `fs/promises` | File I/O and directory creation |
| `path` | Cross-platform path handling |
| `@medusajs/framework` | `AbstractFileProviderService`, `MedusaError` |
