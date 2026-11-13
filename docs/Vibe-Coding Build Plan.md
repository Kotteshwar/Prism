# Vibe-Coding Build Plan: Prism Gallery

**Version:** 2.0  
**Depends on:** PRD v2.0, Architecture v2.0, Data Model v2.0, UI/UX Spec v2.0

---

## Phase 1: Foundation & Setup

### Prompt 1.1: Project Scaffold & Design System
**Context:** We are building a React PWA called Prism, a dark-mode media gallery.
**Task:** 
1. Scaffold a new project using Vite + React + TypeScript.
2. Install dependencies: `react-router-dom`, `idb-keyval`, `lucide-react`, `vite-plugin-pwa`.
3. Install Tailwind CSS and configure `tailwind.config.js` exactly as defined in Document 4, Section 1 (Colors, Fonts, Border Radius).
4. Add the Google Fonts (Epilogue, Inter, Google Sans) and Material Symbols Outlined to `index.html`.
5. Set the default background to `bg-surface` and text to `text-on-surface`.

**Acceptance Criteria:**
- App runs locally with `npm run dev`.
- A blank page displays with a dark grey background (`#131314`) and light text.
- Tailwind classes like `bg-surface-container-low` and `font-headline` work correctly.

---

## Phase 2: Backend Infrastructure (Cloudflare)

### Prompt 2.1: Cloudflare Pages Functions & Auth
**Context:** We need a proxy to Koofr WebDAV because Koofr does not support CORS. We also need an admin passphrase check.
**Task:**
1. Create the `functions/` directory structure as defined in Document 2, Section 3.
2. Implement `functions/api/auth.ts`: A `POST` endpoint that checks a passphrase against `env.ADMIN_PASSPHRASE`. On success, sign an HMAC-SHA256 cookie (`gallery_session`) with `env.SESSION_SECRET` and set it as `HttpOnly`, `Secure`, `SameSite=Lax`. Implement `DELETE` for logout.
3. Implement `functions/_middleware.ts`: Check for the `gallery_session` cookie on all `/api/dav/*` routes. If invalid/missing, return `401`. Add CORS headers for `OPTIONS` requests.

**Acceptance Criteria:**
- `POST /api/auth` with the correct passphrase returns a `Set-Cookie` header.
- `POST /api/auth` with the wrong passphrase returns `401`.
- Requesting `/api/dav/test` without a cookie returns `401`.

### Prompt 2.2: Koofr WebDAV Proxy
**Context:** The proxy must forward requests to Koofr.
**Task:**
1. Implement `functions/api/dav/[[path]].ts`.
2. Allow methods: `PROPFIND`, `GET`, `PUT`, `MKCOL`, `DELETE`, `MOVE`.
3. Inject the `Authorization: Basic base64(env.KOOFR_USER:env.KOOFR_APP_PASSWORD)` header.
4. Forward the request to `https://app.koofr.net/dav/Koofr/{path}`.
5. **Crucial:** For `PUT` requests, stream `request.body` directly to the `fetch` call. Do not buffer.
6. For `MOVE` requests, rewrite the `Destination` header to the full Koofr URL.
7. Return the upstream response unmodified.

**Acceptance Criteria:**
- `PROPFIND /api/dav/` with a valid cookie returns a `207 Multi-Status` XML response from Koofr.
- `GET /api/dav/{path}` with a `Range: bytes=0-1048575` header returns a `206 Partial Content`.

---

## Phase 3: Data Parsing & Local State

### Prompt 3.1: WebDAV Parsing & Local State
**Context:** We need to parse the Koofr XML and manage local favorites/settings.
**Task:**
1. Implement `parsePropfindXml(xmlText)` in `src/lib/webdav.ts` as defined in Document 3. It must skip the first `<d:response>` (the folder itself) and return an array of `MediaItem` objects.
2. Implement `filenameDate(name)` in `src/lib/utils.ts`. It must parse `YYYYMMDD_HHMMSS` and validate the date.
3. Implement the `src/lib/localState.ts` file using `idb-keyval` for `getFavorites`, `toggleFavorite`, `getHidden`, `toggleHidden`, `getSettings`, and `saveSettings`.

**Acceptance Criteria:**
- Passing a sample Koofr XML string to `parsePropfindXml` returns the correct number of media items without the folder entry.
- `filenameDate("20240315_142530.jpg")` returns `"2024-03-15T14:25:30.000Z"`.
- `toggleFavorite` correctly adds and removes paths from IndexedDB.

---

## Phase 4: Main UI Shell

### Prompt 4.1: App Shell & Navigation
**Context:** Build the global layout that wraps all screens.
**Task:**
1. Create `src/components/layout/AppShell.tsx`.
2. Implement the fixed top header (`h-16 px-5`) with the Prism logo and dynamic page title (Photos/Videos/Collections).
3. Implement the floating bottom navigation pill (`rounded-full backdrop-blur-xl bg-surface-container-lowest/80`). Use `lucide-react` icons. Highlight the active route.
4. Set up React Router in `App.tsx` with routes: `/photos`, `/videos`, `/collections`, `/settings`, `/login`.

**Acceptance Criteria:**
- Navigating between routes updates the URL and the active state of the bottom nav.
- The header and bottom nav remain fixed while scrolling content.

---

## Phase 5: Gallery Views

### Prompt 5.1: Photos Timeline View
**Context:** Build the Photos screen (Screens 1 & 8).
**Task:**
1. Create `src/pages/Photos.tsx`.
2. Use the `useMedia('/Photos')` hook (fetches `PROPFIND`, merges local state, sorts by `captureDate`).
3. Group the sorted array by Month/Year.
4. Render the timeline:
   - Sticky month headers (`sticky top-16`).
   - 3-column grid (`grid grid-cols-3 gap-2`).
   - Feature images span 2 columns (`col-span-2 aspect-[4/3]`).
   - Standard images span 1 column (`aspect-square`).
5. Add the floating right-edge timeline scrubber (`OCT`, `SEP`, `AUG`).

**Acceptance Criteria:**
- Photos render in a timeline grouped by month.
- The scrubber scrolls to the correct section when clicked.
- Hidden items are excluded from the grid.

### Prompt 5.2: Video Gallery View
**Context:** Build the Videos screen (Screen 4).
**Task:**
1. Create `src/pages/Videos.tsx`.
2. Use `useMedia('/Videos')`.
3. Render the filter chips (All Videos, 4K, Cinematic, etc.) at the top.
4. Render a 2-column grid:
   - Feature videos (e.g., first video) span 2 columns (`aspect-[16/9]`).
   - Standard videos are square (`aspect-square`).
5. Overlay a central Play button and a duration badge on each video card.

**Acceptance Criteria:**
- Videos display in a 2-column grid with correct aspect ratios.
- Clicking a filter chip (e.g., 4K) filters the visible videos based on metadata.

---

## Phase 6: Detail Views & Metadata Extraction

### Prompt 6.1: Media Processing (EXIF & MP4)
**Context:** Implement the heavy lifting for the Inspector panels.
**Task:**
1. Install `exifreader`, `mp4box`, and `piexifjs`.
2. Implement `extractImageMetadata` in `src/lib/media.ts` using `exifreader` and Document 3, Section 4.2.
3. Implement `extractVideoMetadata` in `src/lib/media.ts` using `mp4box` and Document 3, Section 4.3 (including the 1 MiB Range limit and `fileStart` chunking).

**Acceptance Criteria:**
- Passing a JPEG `ArrayBuffer` returns populated camera metadata.
- Passing an MP4 `ArrayBuffer` returns video specs (width, height, duration, bitrate).

### Prompt 6.2: Photo & Video Detail Screens
**Context:** Build the Lightbox views (Screens 6 & 9).
**Task:**
1. Create `src/pages/PhotoDetail.tsx` and `src/pages/VideoDetail.tsx`.
2. **Photo Detail:** Full-screen image, filmstrip at bottom. Add the "Camera & Optics" and "File Specifications" panels. On mount, call `extractImageMetadata`.
3. **Video Detail:** Native `<video>` player. Add the "Production Rig" and "Deep Video Specifications" panels. On mount, call `extractVideoMetadata`.
4. Bind the `isFavorite` state to the star icon in the top header.

**Acceptance Criteria:**
- Opening a photo displays the image and populates the EXIF data within 1.5s.
- Opening a video allows playback and displays the MP4 metadata.
- Toggling the star updates the IndexedDB state.

---

## Phase 7: Batch Actions & Modals

### Prompt 7.1: Multi-Select Toolbar
**Context:** Implement the batch actions from Screen 3.
**Task:**
1. Create a `useSelection` hook to manage a `Set` of selected paths.
2. Add a long-press handler to the `MediaCell` component to trigger selection mode.
3. Render the top bar replacement ("X Selected", "Select All", "Close").
4. Render the bottom floating toolbar (Share, Add to Album, Favorite, Hide, Delete).
5. Implement actions:
   - *Favorite/Hide:* Toggle in IndexedDB.
   - *Delete:* Call `DELETE /api/dav/{path}`. Update UI optimistically.

**Acceptance Criteria:**
- Long-pressing a photo enters selection mode.
- Clicking "Delete" removes the item from Koofr and the UI immediately.

### Prompt 7.2: Create Album & Upload
**Context:** Implement the Create Album modal (Screen 2).
**Task:**
1. Create `src/components/modals/CreateAlbum.tsx`.
2. Render the bottom sheet UI (Album Name, Destination Path, Dropzone).
3. On submit:
   - Call `MKCOL /api/dav/Albums/{albumName}`.
   - If files are selected, iterate and call `PUT /api/dav/Albums/{albumName}/{filename}`.
4. Show the green "Album Created!" success state.

**Acceptance Criteria:**
- Creating an album creates a new folder in Koofr.
- Selecting files and uploading them correctly places them in the new album.

---

## Phase 8: Sharing & Settings

### Prompt 8.1: Share Sheet & EXIF Stripping
**Context:** Implement the Share modal (Screen 7).
**Task:**
1. Create `src/components/modals/ShareSheet.tsx`.
2. Implement `POST /api/share` (Cloudflare Function) to generate a KV token with a 24-hour TTL.
3. Implement the Share UI: Copy URL, Copy Image, and Social buttons (AirDrop, WhatsApp, etc.).
4. For the social buttons, use `navigator.share({ files: [file] })`.
5. Add the "Preserve EXIF & GPS" toggle. If off, call `stripExifFromBlob` (using `piexifjs` as defined in Document 3, Section 4.4) before passing the file to `navigator.share`.

**Acceptance Criteria:**
- Clicking "Copy URL" generates a share token and copies it.
- Sharing to a native app works.
- Toggling off EXIF removes the metadata from the shared file (verify by sharing to yourself and checking the file properties).

### Prompt 8.2: Settings & PWA
**Context:** Finalize the Settings screen (Screen 11) and PWA capabilities.
**Task:**
1. Create `src/pages/Settings.tsx`.
2. Bind the "Frosted Glass Intensity" slider to a CSS variable (`--glass-blur`) that controls `backdrop-filter`.
3. Bind the "Pure OLED Mode" toggle to a class on `<html>` that switches the background to `#000000`.
4. Configure `vite-plugin-pwa` in `vite.config.ts` as defined in Document 2, Section 8 (cache the shell, NetworkOnly for media).
5. Generate a `manifest.json` with the Prism logo and dark theme colors.

**Acceptance Criteria:**
- Adjusting the slider changes the blurriness of the headers.
- The app can be installed to the home screen as a PWA.
- Media requests bypass the service worker cache.

