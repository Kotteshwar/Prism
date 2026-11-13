# Product Requirements Document: Prism Gallery (Koofr PWA)

**Version:** 2.0  
**Status:** Ready for build  
**Last updated:** 2026-11-12  
**UI Reference:** Stitch HTML exports + `DESIGN.md`

---

## 1. Overview
**Prism** is a single-user, installable web application (PWA) that presents photos and videos stored in **Koofr (via WebDAV)** as a premium, dark-themed gallery. It solves the "upload date vs. capture date" problem by parsing dates from filenames and extracting EXIF/MP4 metadata on demand. The app provides full media management (upload, delete, organize into albums), batch actions, OS-level sharing, and expiring share links, all running on Cloudflare Pages/Workers.

---

## 2. Problem Statement
Koofr’s WebDAV `PROPFIND` response returns the file's modification/upload time, which is useless for chronological browsing. Media needs to be sorted by *capture date* (parsed from filenames like `20240315_142530.jpg`). Furthermore, Koofr lacks a native, visually rich media viewing experience. The user needs a fast, app-like interface to browse, organize, and share media without exposing Koofr credentials.

---

## 3. Goals
| # | Goal | Measure |
|---|---|---|
| G1 | Premium Media Gallery | Match the exact visual language from `DESIGN.md` (Glassmorphism, Epilogue/Inter fonts, dark mode). |
| G2 | Chronological Sorting | Group media by month/year. Sort descending by filename date. |
| G3 | Metadata Extraction | Show deep EXIF (Camera, Aperture, Shutter, ISO) and Video specs (Codec, Bitrate, FPS) on demand. |
| G4 | Full CRUD via WebDAV | Upload, Delete, Favorite, Hide, and Move (to Albums) directly from the app. |
| G5 | Secure Sharing | Generate expiring token links (KV) and use the native OS share sheet (Web Share API). |
| G6 | Zero Cost | Run entirely on free infrastructure (Cloudflare Pages, Workers, KV). |

---

## 4. Non-Goals
- **Search:** Full-text or metadata search is explicitly **removed** from scope.
- **Multi-Cloud:** The app is hardcoded to Koofr WebDAV. (UI elements for S3/NAS in the Settings screen should be disabled or marked "Coming Soon").
- **Multi-tenant OAuth:** Single admin user only.
- **MKV Parsing:** MP4/MOV container metadata only. MKV falls back to filename.

---

## 5. Personas
- **Admin (You):** Single authenticated user. Uploads media, creates albums, manages favorites/hidden files, deletes files, and generates share links.
- **Viewer (Recipient):** Opens an expiring `/share/:token` link. Can view/play specific shared files but cannot browse the rest of the gallery.

---

## 6. Functional Requirements

### FR-1: Authentication & Session
- **FR-1.1:** Admin logs in via `POST /api/auth` with a passphrase. 
- **FR-1.2:** Successful login issues an `HttpOnly`, `Secure`, `SameSite=Lax` cookie with a 7-day expiry.
- **FR-1.3:** Koofr credentials (`KOOFR_USER`, `KOOFR_APP_PASSWORD`) exist only as Cloudflare secrets.

### FR-2: WebDAV Proxy (The Cloudflare Worker)
- **FR-2.1 Allowlist Methods:** The proxy must forward:
  - `PROPFIND` (List files/folders)
  - `GET` (Download media, supports `Range` for metadata)
  - `PUT` (Upload new media)
  - `MKCOL` (Create new album folders)
  - `DELETE` (Delete files)
  - `MOVE` (Rename files / Move to Album)
- **FR-2.2 Path Normalization:** Reject paths with `..`. 
- **FR-2.3 Auth Injection:** The proxy injects `Authorization: Basic base64(user:pass)`.

### FR-3: Galleries (Photos, Videos, Collections)
- **FR-3.1 Sorting & Grouping:** Parse `YYYYMMDD` from filename. Group by Month/Year (e.g., "October 2024"). If no filename date, use `uploadDate`.
- **FR-3.2 Photos View (Screen 8):** Masonry/Timeline grid. Includes a floating right-edge timeline scrubber (OCT, SEP, AUG).
- **FR-3.3 Videos View (Screen 4):** 2-column grid. Specialized aspect ratios for 16:9 and square videos. Shows duration, resolution, and play icon.
- **FR-3.4 Collections View (Screen 5):** Folder grid. Shows "Recent Highlights" hero card. 
- **FR-3.5 Filters:** 
  - *Photos:* Favorites, RAW Only, Portraits, <1/1000s, Locations, Videos.
  - *Videos:* All Videos, 4K Clips, Cinematic, Slow-mo, Timelapses.

### FR-4: Media Detail Views (Lightboxes)
- **FR-4.1 Photo Detail (Screen 9):** Full-screen view. Swipeable mini-filmstrip at the bottom. 
- **FR-4.2 Video Detail (Screen 6):** Custom video player with scrubber, play/pause, mute, fullscreen.
- **FR-4.3 Metadata Inspector (EXIF/MP4):**
  - *Images:* Camera Make/Model, Lens, Aperture (f/1.7), Shutter (1/125s), Focal Length, ISO, File Format, Resolution.
  - *Videos:* Resolution/FPS, Bitrate/Codec, Color Profile (S-Gamut3.Cine), Shutter/ISO, Audio specs, Container format.
  - *Extraction:* Client-side parsing via `ExifReader` (images) and `mp4box.js` (video). Fetches first 1 MiB via `Range`.

### FR-5: Batch Actions (Screen 3)
- **FR-5.1 Multi-Select:** Tap and hold to enter selection mode. Top bar shows "X Selected" and "Select All".
- **FR-5.2 Batch Toolbar:** Bottom floating pill with 5 actions:
  - **Share:** Opens Share sheet.
  - **Add to Album:** Move files (`MOVE` request) to a specific Koofr folder.
  - **Favorite:** Saves to local IndexedDB.
  - **Hide:** Saves to local IndexedDB (removes from timeline).
  - **Delete:** Issues `DELETE` to Koofr. Requires confirmation modal.

### FR-6: Uploads & Albums (Screen 2)
- **FR-6.1 Create Album:** Bottom sheet with Album Name, Destination Path, and a "Same name" toggle. Calls `MKCOL`.
- **FR-6.2 Upload:** Drag-and-drop or file picker. Issues `PUT` to Koofr. 
- **FR-6.3 Auto-Rename:** Before uploading, extract metadata and rename to `YYYYMMDD_HHMMSS_filename.ext`.

### FR-7: Sharing (Screen 7)
- **FR-7.1 Expiring Links:** Generate a token, store in KV with `expirationTtl`. Returns `prism.gallery/s/:token`.
- **FR-7.2 Native OS Share:** Use `navigator.share()` for AirDrop, WhatsApp, Instagram, etc.
- **FR-7.3 EXIF Stripping:** Toggle "Preserve EXIF & GPS". If off, strip EXIF client-side (using `piexifjs`) before sharing/downloading.

### FR-8: Settings (Screen 11)
- **FR-8.1 Profile & Vault:** Display storage usage. (Since Koofr doesn't expose total quota via WebDAV, mock this UI or fetch from a custom endpoint if available. Otherwise, show "Koofr Connected").
- **FR-8.2 Preferences:** 
  - *Glass Intensity Slider:* CSS `backdrop-filter: blur()` variable.
  - *Pure OLED Mode:* Switch background to `#000000`.
  - *Cache Limit:* UI only (browser handles Cache Storage limits).
- **FR-8.3 Log Out:** Clears cookie and redirects to login.

---

## 7. Non-Functional Requirements

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | Design System | Strictly adhere to `DESIGN.md` (Tailwind tokens, Epilogue/Inter fonts, exact hex codes). |
| NFR-2 | Performance | Proxy CPU < 10ms. Metadata probes limited to 1 MiB. |
| NFR-3 | Local State | Use `idb-keyval` for Favorites, Hidden files, and Glass/OLED settings. |
| NFR-4 | Security | Zero credentials in client bundle. Share links must be unguessable (256-bit entropy). |
| NFR-5 | PWA | Cache app shell only. Media is `NetworkOnly`. |

---

## 8. Data Contracts

```ts
interface MediaItem {
  path: string;              // e.g. "/Photos/20240315_142530.jpg"
  name: string;
  type: "image" | "video" | "folder";
  size: number;
  uploadDate: string;        // ISO 8601 from WebDAV
  captureDate: string | null;// ISO 8601 from filename
  isFavorite: boolean;       // from IndexedDB
  isHidden: boolean;         // from IndexedDB
}

interface MediaMetadata {
  captureDate: string | null;
  cameraMake?: string;
  cameraModel?: string;
  lensModel?: string;
  fNumber?: number;
  exposureTime?: string;
  iso?: number;
  focalLength?: number;
  video?: {
    width: number;
    height: number;
    durationSeconds: number;
    codec: string;
    bitrate: number;
    frameRate: number;
  };
}
```

---

## 9. Success Metrics
- App fully matches the provided Stitch designs on a 390px wide mobile viewport.
- Metadata loads within 1.5s of opening the file info panel.
- Deleting or moving a file reflects instantly in the UI (optimistic update).
- Share links expire accurately via KV TTL.