# SpecKit — @medusajs/file-local

---

## 1. Unit specs — `upload`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U1 | Public file upload | `{ filename: "img.png", access: "public", content: <utf8> }` | File written to `upload_dir`; returns `{ key: "<ts>-img.png", url: "<backend_url>/<ts>-img.png" }` |
| U2 | Private file upload | `{ filename: "doc.pdf", access: "private", content: ... }` | File written to `private_upload_dir`; key starts with `"private-"` |
| U3 | Base64 content | Base64-encoded string | Decoded to binary buffer before write |
| U4 | UTF-8 content | Plain UTF-8 string | Stored as UTF-8 |
| U5 | Subdirectory in filename | `{ filename: "receipts/doc.pdf" }` | Directory `receipts/` created; key is `receipts/<ts>-doc.pdf` |
| U6 | Directory auto-created | Target dir does not exist | `ensureDirExists` calls `fs.mkdir({ recursive: true })` |
| U7 | No file argument | `null` | Throws `MedusaError(INVALID_DATA, "No file provided")` |
| U8 | No filename | `{ content: "..." }` | Throws `MedusaError(INVALID_DATA, "No filename provided")` |
| U9 | Key format — public | `access: "public"` | Key matches `<timestamp>-<filename>` |
| U10 | Key format — private | `access: "private"` | Key matches `private-<timestamp>-<filename>` |

---

## 2. Unit specs — `delete`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U11 | Public file delete | `{ fileKey: "12345-img.png" }` | `fs.unlink` called with path in `upload_dir` |
| U12 | Private file delete | `{ fileKey: "private-12345-doc.pdf" }` | `fs.unlink` called with path in `private_upload_dir` |
| U13 | File not found (ENOENT) | File was already deleted | No error thrown; silent no-op |
| U14 | Bulk delete | Array of 2 file objects | Both files deleted |
| U15 | Non-ENOENT error | Permission denied | Error rethrown |

---

## 3. Unit specs — `getPresignedDownloadUrl`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U16 | Public file exists | `{ fileKey: "12345-img.png" }` | Returns `"<backend_url>/12345-img.png"` |
| U17 | Private file exists | `{ fileKey: "private-12345-doc.pdf" }` | Returns `"<backend_url>/private-12345-doc.pdf"` |
| U18 | File does not exist | Non-existent key | Throws `MedusaError(NOT_FOUND, "File with key ... not found")` |

---

## 4. Unit specs — `getPresignedUploadUrl`

| # | Scenario | Input | Expected outcome |
|---|---|---|---|
| U19 | Any filename | `{ filename: "photo.jpg" }` | Returns `{ url: "/admin/uploads", key: "photo.jpg" }` |
| U20 | No filename | `{}` | Throws `MedusaError(INVALID_DATA, "No filename provided")` |

---

## 5. Unit specs — URL construction

| # | Scenario | Expected outcome |
|---|---|---|
| U21 | Default backend_url | `options: {}` | URL starts with `"http://localhost:9000/static/"` |
| U22 | Custom backend_url | `backend_url: "http://myapp.local/files"` | URL starts with `"http://myapp.local/files/"` |
| U23 | Key with subdirectory | `fileKey: "receipts/file.pdf"` | Full URL includes `receipts/file.pdf` path |

---

## 6. Integration specs

| # | Scenario | Expected outcome |
|---|---|---|
| I1 | Upload then serve | Upload file; GET `<backend_url>/<key>` → 200 with correct content |
| I2 | Upload then delete | Delete file; `getPresignedDownloadUrl` → `MedusaError(NOT_FOUND)` |
| I3 | Upload stream | Pipe stream → `promise` resolves with `{ url, key }` |

---

## 7. Acceptance criteria

- Private file keys always start with `"private-"`.
- `ENOENT` on delete is silently ignored.
- File content is never corrupted regardless of encoding (base64 / UTF-8 / binary).
- Missing directories are auto-created on first upload.
- `getPresignedUploadUrl` always returns `url: "/admin/uploads"` (no client-to-disk bypass).
