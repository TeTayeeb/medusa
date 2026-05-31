# @medusajs/file-local

Local filesystem file storage provider for Medusa v2. Implements `IFileProvider` via `AbstractFileProviderService` and registers under `Modules.FILE` with identifier `localfs`.

> ⚠️ **Development only.** This provider stores files on the local disk and serves them via a static URL. It has no access control beyond path separation and is **not suitable for production deployments**.

## Installation

```bash
npm install @medusajs/file-local
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
            resolve: "@medusajs/file-local",
            id: "local",
            options: {
              upload_dir: "static",                      // Directory to write public files
              backend_url: "http://localhost:9000/static", // Base URL for serving files
            },
          },
        ],
      },
    },
  ],
})
```

### Options reference

| Option | Type | Default | Description |
|---|---|---|---|
| `upload_dir` | `string` | `<cwd>/static` | Directory for public file storage |
| `private_upload_dir` | `string` | `<cwd>/static` | Directory for private file storage (same as public by default — see note below) |
| `backend_url` | `string` | `http://localhost:9000/static` | Base URL used to construct file URLs |

> **Private files note**: Because this provider serves files via a static endpoint, private files placed in `static/` are accessible to anyone who knows the filename. Private file URLs are prefixed with `private-` to distinguish them from public files but are not truly access-controlled.

## File key format

All files are keyed as:
```
[subdirPath/]<prefix><timestamp>-<originalFilename>
```

- Public files: no prefix (`Date.now()-filename.ext`)
- Private files: `private-Date.now()-filename.ext`

The `private-` prefix is used internally to determine the correct base directory for delete and presigned URL operations.

## Key operations

### `upload(file)`
- Creates the target directory recursively if it does not exist.
- Detects base64 content; falls back to UTF-8, then binary.
- Writes to `<upload_dir>/<fileKey>` using `fs.promises.writeFile`.
- Returns `{ key, url: "<backend_url>/<fileKey>" }`.

### `delete(files)`
- Checks whether `fileKey` starts with `"private-"` to pick the correct base directory.
- Calls `fs.promises.unlink(filePath)`.
- Silently ignores `ENOENT` (file already deleted); rethrows other errors.

### `getPresignedDownloadUrl(fileData)`
- Verifies the file exists with `fs.promises.access`.
- Returns `backend_url + "/" + fileKey` (a direct static URL, not a time-limited presigned URL).

### `getPresignedUploadUrl(fileData)`
- Returns `{ url: "/admin/uploads", key: filename }`.
- The Medusa admin upload endpoint handles the actual write — no client-to-disk bypass is possible.

### `getUploadStream(fileData)`
Returns a `WriteStream` wrapping `fs.createWriteStream`, with a `Promise` that resolves on `finish`.

### `getDownloadStream(file)` / `getAsBuffer(file)`
Returns a `ReadStream` or `Buffer` from the local filesystem.

## Serving static files

Files in `upload_dir` should be served by Express static middleware. In a default Medusa setup this is configured automatically when using the local provider:

```ts
// Medusa serves /static → <cwd>/static
app.use("/static", express.static(path.join(process.cwd(), "static")))
```

## Differences vs file-s3

| Feature | file-s3 | file-local |
|---|---|---|
| Storage | Amazon S3 | Local disk |
| Key uniqueness | ULID | `Date.now()` timestamp |
| Private file access control | S3 bucket policies | None (path-based only) |
| Presigned upload | Direct to S3 | Via Medusa `/admin/uploads` |
| Suitable for production | ✅ | ❌ |
