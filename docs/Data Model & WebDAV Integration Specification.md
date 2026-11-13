# Data Model & WebDAV Integration Specification

**Version:** 2.0  
**Depends on:** PRD v2.0, Technical Architecture v2.0

---

## 1. Core TypeScript Types (Paste Into `src/types/prism.ts`)

```ts
// ── Domain Types ──────────────────────────────────────────────

export type MediaType = "image" | "video" | "folder";

export interface MediaItem {
  /** Normalized WebDAV path, always starts with "/" */
  path: string;
  /** Basename including extension */
  name: string;
  type: MediaType;
  mimeType: string;
  size: number; // bytes
  uploadDate: string; // ISO 8601 UTC from WebDAV
  captureDate: string | null; // ISO 8601 UTC from filename
  needsRename: boolean;
  etag?: string;
  
  // Merged from IndexedDB
  isFavorite: boolean;
  isHidden: boolean;
}

export interface MediaMetadata {
  captureDate: string | null;
  source: "filename" | "exif" | "mp4" | "none";
  cameraMake?: string;
  cameraModel?: string;
  lensModel?: string;
  fNumber?: number;
  exposureTime?: string;
  iso?: number;
  focalLength?: number;
  gps?: { lat: number; lng: number };
  video?: {
    width: number;
    height: number;
    durationSeconds: number;
    codec: string;
    bitrate: number;
    frameRate: number;
  };
  warning?: string;
}

export interface AppSettings {
  glassIntensity: number; // 0-100
  oledMode: boolean;
}
```

---

## 2. Local State Management (IndexedDB Spec)

Use `idb-keyval` for lightweight local storage. Create `src/lib/localState.ts`:

```ts
import { get, set } from 'idb-keyval';

const FAVORITES_KEY = 'prism_favorites';
const HIDDEN_KEY = 'prism_hidden';
const SETTINGS_KEY = 'prism_settings';

// ── Favorites ────────────────────────────────────────────────
export async function getFavorites(): Promise<string[]> {
  return (await get<string[]>(FAVORITES_KEY)) || [];
}

export async function toggleFavorite(path: string): Promise<boolean> {
  const favs = await getFavorites();
  const index = favs.indexOf(path);
  let isFav = false;
  if (index > -1) {
    favs.splice(index, 1);
  } else {
    favs.push(path);
    isFav = true;
  }
  await set(FAVORITES_KEY, favs);
  return isFav;
}

// ── Hidden Files ─────────────────────────────────────────────
export async function getHidden(): Promise<string[]> {
  return (await get<string[]>(HIDDEN_KEY)) || [];
}

export async function toggleHidden(path: string): Promise<boolean> {
  const hidden = await getHidden();
  const index = hidden.indexOf(path);
  let isHidden = false;
  if (index > -1) {
    hidden.splice(index, 1);
  } else {
    hidden.push(path);
    isHidden = true;
  }
  await set(HIDDEN_KEY, hidden);
  return isHidden;
}

// ── Settings ─────────────────────────────────────────────────
export async function getSettings(): Promise<AppSettings> {
  return (await get<AppSettings>(SETTINGS_KEY)) || { glassIntensity: 75, oledMode: false };
}

export async function saveSettings(settings: AppSettings): Promise<void> {
  await set(SETTINGS_KEY, settings);
}
```

---

## 3. WebDAV Integration Spec (Proxy Routes)

All requests go to `/api/dav/*`. The Cloudflare Worker forwards them to `https://app.koofr.net/dav/Koofr/*`.

### 3.1 List Directory (`PROPFIND`)
*   **Method:** `PROPFIND`
*   **Headers:** `Depth: 1`, `Content-Type: application/xml`
*   **Body:**
    ```xml
    <?xml version="1.0" encoding="utf-8" ?>
    <d:propfind xmlns:d="DAV:">
      <d:prop>
        <d:getlastmodified/>
        <d:getcontentlength/>
        <d:getcontenttype/>
        <d:getetag/>
        <d:resourcetype/>
      </d:prop>
    </d:propfind>
    ```
*   **Client Parser:** `parsePropfindXml(xmlText)` → `MediaItem[]`. Skip the first response (the folder itself). Decode `href`. Apply `filenameDate()`.

### 3.2 Upload File (`PUT`)
*   **Method:** `PUT`
*   **Headers:** `Content-Type: <file.mime>`
*   **Body:** `File` or `Blob` directly.
*   **Note:** Must use streaming on the Worker side (`body: request.body`).

### 3.3 Create Album Folder (`MKCOL`)
*   **Method:** `MKCOL`
*   **Body:** None.
*   **Example:** `MKCOL /api/dav/Albums/Summer%20in%20Big%20Sur`

### 3.4 Delete File (`DELETE`)
*   **Method:** `DELETE`
*   **Body:** None.
*   **Example:** `DELETE /api/dav/Photos/20240315_142530.jpg`

### 3.5 Move / Rename File (`MOVE`)
*   **Method:** `MOVE`
*   **Headers:** `Destination: /api/dav/{newPath}`, `Overwrite: F`
*   **Example:** `MOVE /api/dav/Inbox/IMG_0042.MP4` with `Destination: /api/dav/Photos/20240315_142530_IMG_0042.MP4`

---

## 4. Client-Side Media Processing

### 4.1 Fetching Metadata Probe (1 MiB Limit)
```ts
export async function fetchMetadataProbe(path: string): Promise<ArrayBuffer> {
  const res = await fetch(`/api/dav${path}`, {
    headers: { Range: "bytes=0-1048575" },
  });
  if (!res.ok) throw new Error(`Probe failed: ${res.status}`);
  return res.arrayBuffer();
}
```

### 4.2 Image EXIF Parsing (`ExifReader`)
```ts
import ExifReader from 'exifreader';

export async function extractImageMetadata(buffer: ArrayBuffer): Promise<MediaMetadata> {
  try {
    const tags = ExifReader.load(buffer);
    const rawDate = tags.DateTimeOriginal?.description || tags.DateTime?.description;
    let captureDate = null;
    
    if (rawDate) {
      // Convert "2024:03:15 14:25:30" to ISO
      const m = rawDate.match(/^(\d{4}):(\d{2}):(\d{2}) (\d{2}):(\d{2}):(\d{2})/);
      if (m) captureDate = `${m[1]}-${m[2]}-${m[3]}T${m[4]}:${m[5]}:${m[6]}.000Z`;
    }

    return {
      captureDate,
      source: "exif",
      cameraMake: tags.Make?.description,
      cameraModel: tags.Model?.description,
      lensModel: tags.LensModel?.description,
      fNumber: tags.FNumber?.value,
      exposureTime: tags.ExposureTime?.description,
      iso: tags.ISOSpeedRatings?.value,
      focalLength: tags.FocalLength?.value,
    };
  } catch (e) {
    return { captureDate: null, source: "none", warning: "EXIF parse failed" };
  }
}
```

### 4.3 Video MP4 Parsing (`mp4box.js`)
```ts
import { createFile } from 'mp4box';

export async function extractVideoMetadata(buffer: ArrayBuffer): Promise<MediaMetadata> {
  return new Promise((resolve) => {
    const file = createFile();
    let settled = false;
    
    const finish = (meta: MediaMetadata) => {
      if (settled) return;
      settled = true;
      resolve(meta);
    };

    // 10-second timeout
    setTimeout(() => finish({ captureDate: null, source: "none", warning: "moov not found in 1MiB" }), 10000);

    file.onReady = (info) => {
      const videoTrack = info.tracks?.find((t: any) => t.video);
      finish({
        captureDate: info.created || null,
        source: "mp4",
        video: videoTrack ? {
          width: videoTrack.track_width,
          height: videoTrack.track_height,
          durationSeconds: info.duration / info.timescale,
          codec: videoTrack.codec,
          bitrate: videoTrack.bitrate,
          frameRate: videoTrack.nb_samples / (info.duration / info.timescale) // Approx FPS
        } : undefined,
      });
    };

    // Feed chunks to mp4box.js
    const CHUNK_SIZE = 65536;
    const view = new Uint8Array(buffer);
    for (let offset = 0; offset < view.byteLength; offset += CHUNK_SIZE) {
      const slice = view.slice(offset, offset + CHUNK_SIZE).buffer;
      (slice as any).fileStart = offset; // CRITICAL for mp4box
      file.appendBuffer(slice);
    }
    file.flush();
  });
}
```

### 4.4 EXIF Stripping for Sharing (`piexifjs`)
When the user toggles "Preserve EXIF & GPS" off, strip the EXIF before sharing.

```ts
import piexif from 'piexifjs';

export async function stripExifFromBlob(imageBlob: Blob): Promise<File> {
  // piexif requires a Data URL (Base64)
  const reader = new FileReader();
  const dataUrl = await new Promise<string>((resolve) => {
    reader.onload = () => resolve(reader.result as string);
    reader.readAsDataURL(imageBlob);
  });

  // Remove EXIF
  const strippedDataUrl = piexif.remove(dataUrl);

  // Convert back to Blob
  const res = await fetch(strippedDataUrl);
  const strippedBlob = await res.blob();

  return new File([strippedBlob], "image.jpg", { type: "image/jpeg" });
}
```

---

## 5. Filename Parsing & Rename Logic

### 5.1 Date Pattern Matcher
```ts
export function filenameDate(name: string): string | null {
  const m = name.match(/(\d{4})(\d{2})(\d{2})(?:[_-](\d{2})(\d{2})(\d{2}))?/);
  if (!m) return null;
  const [, y, mo, d, h = "00", mi = "00", s = "00"] = m;
  
  // Validate calendar correctness
  if (+mo < 1 || +mo > 12 || +d < 1 || +d > 31) return null;
  const iso = `${y}-${mo}-${d}T${h}:${mi}:${s}.000Z`;
  return isNaN(Date.parse(iso)) ? null : iso;
}
```

### 5.2 Build Rename Target
```ts
export function buildRenameTarget(originalName: string, metadataDate: string): string {
  const date = new Date(metadataDate);
  const stamp = date.toISOString().replace(/[-:T]/g, "").slice(0, 14); // YYYYMMDDHHMMSS
  const dot = originalName.lastIndexOf(".");
  const base = dot === -1 ? originalName : originalName.slice(0, dot);
  const ext = dot === -1 ? "" : originalName.slice(dot);
  return `${stamp}_${base}${ext}`;
}
```

---

## 6. Error Handling Types

```ts
export class WebDavError extends Error {
  constructor(public status: number, public path: string) {
    super(`WebDAV Error ${status} at ${path}`);
    this.name = "WebDavError";
  }
}

export class MetadataParseError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "MetadataParseError";
  }
}
```
