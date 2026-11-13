# Technical Architecture: Prism Gallery PWA

**Version:** 2.0  
**Depends on:** PRD v2.0  
**Deployment target:** Cloudflare Pages + Pages Functions + Workers KV + IndexedDB (Client)

---

## 1. Architecture Principles

1. **Credentials Never Reach the Browser:** Koofr WebDAV passwords and the Admin passphrase exist only as Cloudflare Environment Secrets. The browser holds a signed, `HttpOnly` session cookie.
2. **Thin Proxy, Heavy Client:** Cloudflare Pages Functions have a strict **10 ms CPU limit** per request. The proxy only handles routing, auth injection, and simple path validation. All heavy lifting (XML parsing, EXIF extraction, MP4 parsing, EXIF stripping) happens in the browser.
3. **Local-First UI State:** Features like Favorites, Hidden files, and UI preferences (Glass intensity, OLED mode) are stored in the browser's **IndexedDB**. They do not require a backend database.
4. **Streaming for Uploads:** Uploads (`PUT`) must stream the request body directly to Koofr. Do not buffer files in Worker memory, which will cause out-of-memory errors for large videos.

---

## 2. Overall Topology

```text
┌──────────────────────────────────────────────────────────────┐
│  Browser (React PWA)                                         │
│  - Static assets served by Pages CDN                         │
│  - Metadata & EXIF stripping runs on the client              │
│  - Local state (Favorites, Settings) in IndexedDB            │
└──────────────────────┬───────────────────────────────────────┘
                       │ fetch("/api/...") / navigator.share()
┌──────────────────────▼───────────────────────────────────────┐
│  Cloudflare Pages Functions (Workers runtime)                │
│                                                              │
│  /api/auth          POST  → verify passphrase, set cookie    │
│  /api/dav/*         ANY   → forward to Koofr WebDAV          │
│  /api/share         POST  → generate KV token                │
│  /api/share/:t/*    GET   → restricted byte proxy            │
└──────┬───────────────────────────────┬───────────────────────┘
       │                               │
┌──────▼──────┐                 ┌──────▼──────┐
│ Koofr WebDAV│                 │ Workers KV  │
│ (CRUD)      │                 │ share:*     │
└─────────────┘                 └─────────────┘
```

---

## 3. Pages Functions File Structure

Cloudflare Pages uses file-based routing. The project structure must be:

```text
functions/
  _middleware.ts                 # Global CORS and auth gatekeeper
  api/
    auth.ts                      # POST /api/auth (Login/Logout)
    dav/
      [[path]].ts                # ANY /api/dav/* → Koofr WebDAV proxy
    share/
      index.ts                   # POST /api/share (Create token)
      [token]/
        [[path]].ts              # GET /api/share/:token/* → View token
lib/
  koofr.ts                       # Auth header builder, Path normalization
  session.ts                     # HMAC cookie sign/verify
  share.ts                       # KV schema, Token generation
  cors.ts                        # CORS headers
```

---

## 4. Koofr WebDAV Proxy (`/api/dav/*`)

This is the core of the backend. It must support **all methods** required by the UI.

### 4.1 Method Allowlist
The proxy accepts the following methods and forwards them to `https://app.koofr.net/dav/Koofr/`:

| Method | Purpose | Implementation Note |
|---|---|---|
| `PROPFIND` | List files / Get metadata | Returns XML. Client parses this. |
| `GET` | Download / Stream media | Must pass through `Range` headers exactly. |
| `PUT` | Upload media | **Must pass `request.body` directly to `fetch()`** to stream the file without buffering. |
| `MKCOL` | Create Album folder | No body required. |
| `DELETE` | Delete media | No body. |
| `MOVE` | Move / Rename | Require `Destination` header. Use `Overwrite: F`. |

### 4.2 Path Normalization
- Decode URI components.
- Reject paths containing `..` (Directory traversal prevention).
- Ensure paths start with `/`.
- Attach `Authorization: Basic base64(env.KOOFR_USER:env.KOOFR_APP_PASSWORD)`.

### 4.3 Range Requests and Cloudflare Caching
When extracting EXIF/MP4 data, the client sends `Range: bytes=0-1048575` (1 MiB).
Cloudflare's edge may align origin ranges to 1 MiB boundaries. To avoid double-fetching, the client always requests exactly 1 MiB for metadata probes. The proxy forwards the `Range` header without modification.

---

## 5. Local State Management (IndexedDB)

Because Koofr WebDAV does not natively support custom tags like "Favorites", we store this locally. Use **`idb-keyval`** (a lightweight 600-byte wrapper for IndexedDB).

### 5.1 Database Schema
```ts
// Key: "favorites", Value: string[] (Array of WebDAV paths)
// Key: "hidden", Value: string[] (Array of WebDAV paths)
// Key: "settings", Value: { glassIntensity: number, oledMode: boolean }
```

### 5.2 UI Integration
- When rendering the Photos grid, fetch `favorites` from IndexedDB. Merge it into the `MediaItem` objects. Show the gold star (as seen in Screen 1).
- When toggling a favorite, update IndexedDB, then update local React state.
- **Filtering:** The "Favorites" chip (Screen 1) and "Hide" action (Screen 3) filter against this local array.

---

## 6. Client-Side Media Processing

All heavy computations happen in the React app.

### 6.1 Metadata Extraction (EXIF & MP4)
- **Images:** Use `ExifReader` on a `Range` GET response (`ArrayBuffer`).
- **Videos:** Use `mp4box.js` on a `Range` GET response. Append `ArrayBuffer` chunks. Read `info.created`, `info.duration`, and the `videoTracks[0]` properties.
- **Stitching:** The UI (Screen 9) maps these values to the "Camera & Optics" and "Deep Video Specifications" panels. If a field is missing, display "N/A".

### 6.2 EXIF Stripping (Share Sheet - Screen 7)
When the user toggles "Preserve EXIF & GPS" **OFF**:
1. Fetch the full image `Blob` from `/api/dav/...`.
2. Pass the `ArrayBuffer` to `piexifjs` to remove the EXIF segment.
3. Convert the stripped buffer back to a `Blob`.
4. Pass this sanitized `Blob` to `navigator.share({ files: [new File([blob], "image.jpg")] })`.

If "Preserve EXIF" is **ON**, pass the raw `Blob` directly to `navigator.share()`.

---

## 7. Share Tokens & Viewer Mode

### 7.1 KV Schema
```ts
// Key: `share:${token}` (32-byte base64url string)
interface ShareRecord {
  type: "files";
  paths: string[]; // Exact WebDAV paths
  createdAt: string;
  label?: string;
}
```
TTL is set via `expirationTtl` in KV (minimum 60s, max 30 days).

### 7.2 Share Creation (`POST /api/share`)
- Validates the paths.
- Generates a 256-bit token.
- Writes to KV with TTL.
- Returns `https://prism.gallery/share/${token}`.

### 7.3 Share Viewer (`GET /api/share/:token/*`)
- Extracts `token` and `filePath`.
- Checks KV. If missing → `404`.
- Checks if `filePath` exactly matches an entry in `ShareRecord.paths`. If not → `403`.
- Fetches from Koofr **without** requiring the Admin cookie.

---

## 8. PWA and Service Worker Strategy

Use `vite-plugin-pwa`.

```ts
VitePWA({
  registerType: "autoUpdate",
  workbox: {
    navigateFallback: "/index.html",
    // Prevent the SW from hijacking API and Share routes
    navigateFallbackDenylist: [/^\/api\//, /^\/share\//],
    globPatterns: ["**/*.{js,css,html,svg,png,woff2}"],
    runtimeCaching: [
      {
        // Media files must ALWAYS hit the network
        urlPattern: /\/api\/dav\/.*/,
        handler: "NetworkOnly",
      },
      {
        urlPattern: /\/api\/share\/.*/,
        handler: "NetworkOnly",
      },
    ],
  },
})
```

---

## 9. Deployment Configuration (`wrangler.toml`)

```toml
name = "prism-gallery"
compatibility_date = "2026-11-01"
pages_build_output_dir = "dist"

[[kv_namespaces]]
binding = "SHARE_KV"
id = "<production-kv-id>"

[vars]
APP_ORIGIN = "https://prism.gallery"
KOOFR_USER = "your-email@example.com"
# KOOFR_APP_PASSWORD, ADMIN_PASSPHRASE, SESSION_SECRET must be set via `wrangler pages secret put`
```

---

## 10. Traceability Matrix (PRD v2.0 to Architecture)

| PRD Requirement | Architecture Implementation |
|---|---|
| **FR-2.1** Proxy Allowlist | Section 4.1 (Supports PROPFIND, GET, PUT, DELETE, MKCOL, MOVE) |
| **FR-2.2** Path Normalization | Section 4.2 (Reject `..`) |
| **FR-4.3** Metadata Inspector | Section 6.1 (ExifReader / mp4box.js client-side) |
| **FR-5.2** Batch Actions | Section 5 (IndexedDB for Fav/Hide) + Section 4.1 (`DELETE`, `MOVE`) |
| **FR-6.1** Create Album | Section 4.1 (`MKCOL`) |
| **FR-6.2** Uploads | Section 4.1 (`PUT` streaming) |
| **FR-7.2** Native OS Share | Section 6.2 (`navigator.share`) |
| **FR-7.3** EXIF Stripping | Section 6.2 (`piexifjs`) |
| **FR-8.2** Preferences | Section 5.1 (IndexedDB `settings`) |
