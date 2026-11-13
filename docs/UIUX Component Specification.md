# UI/UX Component Specification: Prism Gallery

**Version:** 2.0  
**Depends on:** PRD v2.0, Architecture v2.0, `DESIGN.md`  
**Tech Stack:** React, Tailwind CSS, React Router, `idb-keyval`, `lucide-react` (or Material Symbols)

---

## 1. Design System Setup (Tailwind)

Instruct the AI to configure `tailwind.config.js` using the exact tokens from your `DESIGN.md` and Stitch exports.

```javascript
module.exports = {
  darkMode: "class",
  content: ["./src/**/*.{js,ts,jsx,tsx}"],
  theme: {
    extend: {
      colors: {
        surface: "#131314",
        "surface-dim": "#131314",
        "surface-bright": "#393939",
        "surface-container-lowest": "#0e0e0e",
        "surface-container-low": "#1b1c1c",
        "surface-container": "#1f2020",
        "surface-container-high": "#2a2a2a",
        "surface-container-highest": "#353535",
        "on-surface": "#e4e2e2",
        "on-surface-variant": "#c4c7ca",
        outline: "#8e9194",
        "outline-variant": "#44474a",
        primary: "#e5eaef",
        "on-primary": "#2b3135",
        "primary-container": "#c9ced3",
        tertiary: "#f8e6db",
        "tertiary-container": "#dbcabf",
        error: "#ffb4ab",
        "error-container": "#93000a",
      },
      fontFamily: {
        headline: ["Epilogue", "sans-serif"],
        body: ["Inter", "sans-serif"],
        label: ["Google Sans", "sans-serif"],
      },
      borderRadius: {
        DEFAULT: "0.25rem",
        lg: "0.5rem",
        xl: "0.75rem",
        "2xl": "1rem",
        full: "9999px",
      },
    },
  },
  plugins: [],
};
```

---

## 2. Global App Shell

Create a layout wrapper `src/components/layout/AppShell.tsx`.

*   **Header:** Fixed top, glassmorphic (`bg-surface/70 backdrop-blur-xl`). Left: Prism Logo + dynamic page title (Photos/Videos/Collections). Right: Profile avatar linking to `/settings`.
*   **Bottom Navigation:** Floating pill at bottom center. Glassmorphic (`bg-surface-container-lowest/80 backdrop-blur-xl`). 3 tabs: Photos (`photo_library`), Collections (`folder_copy`), Videos (`movie`). Active tab gets `text-primary` and FILL 1 icon.
*   **Main Container:** `px-4 pt-16 pb-28 bg-surface min-h-screen`.

---

## 3. Screen-by-Screen Component Breakdown

### 3.1 Photos Timeline (Screens 1, 8, 10)
**Route:** `/photos`
**Component:** `src/pages/Photos.tsx`

*   **Search Bar (Deprecated):** Per the PRD update, remove the search input. Replace it with a simple "Filter" button (`tune` icon) and "Sync" status indicator.
*   **Timeline Scrubber (Screen 8):** Fixed right-side floating glass pill. Links to `#month-oct`, `#month-sep`, etc. Use `scroll-behavior: smooth`.
*   **Timeline Groups:** Map over grouped data.
    *   **Sticky Month Header:** `sticky top-16 z-30 bg-surface-container-lowest/80 backdrop-blur-md rounded-xl p-2`.
    *   **Date Clusters:** Flex row with date badge (`Oct 24`) and subtitle.
    *   **Grid:** `grid grid-cols-3 gap-2`.
    *   **Media Cells:** 
        *   Feature images span 2 columns (`col-span-2 aspect-[4/3]`).
        *   Standard images span 1 column (`aspect-square` or `aspect-[3/4]`).
        *   Hover effects: `group-hover:scale-105 transition-transform`.
        *   Overlays: Gradient bottom scrim, "Fav" star (if `isFavorite`), Location tag.
*   **State:** Fetch folders `/Photos`, parse XML, merge with IndexedDB `favorites` and `hidden`. Filter out `hidden` items. Sort by `captureDate` desc.

### 3.2 Videos Gallery (Screen 4)
**Route:** `/videos`
**Component:** `src/pages/Videos.tsx`

*   **Header:** "LIBRARY" + total video count and size.
*   **Filter Chips:** Horizontal scroll: All Videos, 4K Clips, Cinematic, Slow-mo, Timelapses.
*   **Grid:** `grid grid-cols-2 gap-2.5`.
*   **Feature Card (16:9):** `col-span-2 aspect-[16/9]`. Includes center Play button, bottom left title/location, bottom right duration badge, top badges (4K HDR, ProRes).
*   **Standard Card (Square):** `aspect-square`. Play button overlay, duration badge.
*   **State:** Fetch `/Videos`, extract MP4 thumbnails (using video poster or `<video preload="metadata">`), parse duration.

### 3.3 Collections (Screen 5)
**Route:** `/collections`
**Component:** `src/pages/Collections.tsx`

*   **Hero Card:** "Recent Highlights" spanning full width. Background image with heavy gradient overlay.
*   **Album Grid:** `grid grid-cols-1 sm:grid-cols-2 gap-4`.
*   **Album Card:** 
    *   Mini Collage: 3 slots (1 large left, 2 stacked right).
    *   Info: Folder icon, Name, Last modified date, Item count badge.
*   **New Album Card:** Dashed SVG border (`stroke-dasharray="6, 6"`). Plus icon. Links to `/collections/new`.
*   **State:** Fetch folders matching `/Albums` or root structure.

### 3.4 Create Album Modal (Screen 2)
**Route:** Modal overlay on `/collections`
**Component:** `src/components/modals/CreateAlbum.tsx`

*   **Layout:** Bottom sheet, `rounded-t-[1.5rem]`.
*   **Fields:** 
    *   Album Name (input).
    *   Destination Path (read-only, syncs with name).
    *   Storage Location (locked to Koofr).
    *   Upload Dropzone (`add_photo_alternate` icon).
*   **Action:** "Create" button. On click:
    1. Calls `MKCOL /api/dav/Albums/{albumName}`.
    2. If files selected, calls `PUT` for each file.
    3. Transitions to success state (`bg-emerald-500`).

### 3.5 Multi-Select & Batch Actions (Screen 3)
**Component:** `src/components/batch/BatchToolbar.tsx`

*   **Trigger:** Long-press or click "Select" on any media cell.
*   **Top Bar:** Replaces header. "X Selected", Close (X) button, "Select All" button.
*   **Grid Overlay:** `bg-primary/20 backdrop-brightness-90` + checkmark circle on selected items.
*   **Bottom Floating Toolbar:** 
    *   Buttons: Share, Add to Album, Favorite, Hide, Delete (red text).
    *   **Actions:** 
        *   *Favorite/Hide:* Calls `toggleFavorite` / `toggleHidden` in IndexedDB.
        *   *Delete:* Shows `window.confirm`. Calls `DELETE /api/dav/{path}`.
        *   *Add to Album:* Opens a folder picker modal. Calls `MOVE /api/dav/{path}`.

### 3.6 Photo Detail & Inspector (Screen 9)
**Route:** `/photos/:id`
**Component:** `src/pages/PhotoDetail.tsx`

*   **Top Overlay:** Date, Time, Location. Action pill (Star, Magic wand).
*   **Image Stage:** `aspect-[4/5]`. `cursor-zoom-in` (click to scale 1.4x).
*   **Filmstrip:** Horizontal scroll of nearby photos (`w-12 h-12`).
*   **EXIF Inspector:** `bg-surface-container` card.
    *   Filename, Date, Format badge.
    *   Camera & Optics block (Make, Model, Lens).
    *   2x2 Metric Grid (Aperture, Shutter, Focal, ISO).
    *   File Specs (Resolution, Size, Color profile).
*   **State:** On mount, run `extractImageMetadata()`. If filename date exists, show immediately, then update with EXIF data.

### 3.7 Video Detail & Inspector (Screen 6)
**Route:** `/videos/:id`
**Component:** `src/pages/VideoDetail.tsx`

*   **Video Player:** Custom controls (scrubber, time, play/pause, mute, fullscreen). Use native `<video>` tag.
*   **Filename Card:** Shows size, format badge (MOV/MP4).
*   **Production Rig:** (Mocked/Template data for camera/audio rig).
*   **Video Specs Grid:** Resolution & FPS, Bitrate & Codec, Color & Gamma, Shutter/ISO.
*   **Deep Specs:** Container, Color Depth, Audio Track, Duration, Bitrate Strategy.
*   **State:** On mount, run `extractVideoMetadata()`. Populate the specs.

### 3.8 Share Sheet (Screen 7)
**Component:** `src/components/modals/ShareSheet.tsx`

*   **Layout:** Bottom sheet.
*   **Asset Preview:** Thumbnail, filename, size, format.
*   **Quick Actions:** Copy URL, Copy Image (uses `navigator.clipboard`).
*   **Share Tray:** AirDrop, WhatsApp, Instagram, Snapchat (uses `navigator.share`).
*   **EXIF Toggle:** "Preserve EXIF & GPS". If off, trigger `stripExifFromBlob()` before calling `navigator.share()`.
*   **State:** Creates KV token via `POST /api/share`. Populates the `Copy URL` text.

### 3.9 Settings (Screen 11)
**Route:** `/settings`
**Component:** `src/pages/Settings.tsx`

*   **Segment Pills:** Profile, Storage, Preferences, Security.
*   **Profile Card:** Avatar, Name, Email/ID.
*   **Storage Card:** Progress bar (static/mock data for now, or calculate from `PROPFIND` sizes if possible). "Connect Cloud/NAS" button (disabled/tooltip "Coming Soon").
*   **Preferences Card:**
    *   *Frosted Glass Intensity:* Range slider. On change, update IndexedDB `settings` and CSS `--glass-blur` variable.
    *   *Pure OLED Mode:* Toggle. On change, update `settings` and toggle `.oled` class on `<html>`.
*   **Log Out:** Clears cookie, redirects to `/login`.

---

## 4. State Management & Custom Hooks

Implement global state using React Context or Zustand.

### 4.1 `useMedia()` Hook
```ts
// src/hooks/useMedia.ts
import { useState, useEffect } from 'react';
import { parsePropfindXml } from '../lib/webdav';
import { getFavorites, getHidden } from '../lib/localState';

export function useMedia(folderPath: string) {
  const [items, setItems] = useState<MediaItem[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function load() {
      setLoading(true);
      // 1. Fetch WebDAV PROPFIND
      const res = await fetch(`/api/dav${folderPath}`, {
        method: 'PROPFIND',
        headers: { Depth: '1' },
        body: '<?xml version="1.0"?><d:propfind xmlns:d="DAV:"><d:prop><d:getlastmodified/><d:getcontentlength/><d:getcontenttype/><d:resourcetype/></d:prop></d:propfind>'
      });
      const xml = await res.text();
      const parsed = parsePropfindXml(xml);

      // 2. Merge Local State
      const [favs, hidden] = await Promise.all([getFavorites(), getHidden()]);
      const merged = parsed
        .filter(item => !hidden.includes(item.path))
        .map(item => ({
          ...item,
          isFavorite: favs.includes(item.path),
          isHidden: false
        }));

      // 3. Sort by captureDate desc
      merged.sort((a, b) => new Date(b.captureDate || b.uploadDate).getTime() - new Date(a.captureDate || a.uploadDate).getTime());
      
      setItems(merged);
      setLoading(false);
    }
    load();
  }, [folderPath]);

  return { items, loading };
}
```

### 4.2 `useSettings()` Hook
```ts
// src/hooks/useSettings.ts
import { useState, useEffect } from 'react';
import { getSettings, saveSettings, AppSettings } from '../lib/localState';

export function useSettings() {
  const [settings, setSettings] = useState<AppSettings>({ glassIntensity: 75, oledMode: false });

  useEffect(() => {
    getSettings().then(setSettings);
  }, []);

  const updateSettings = async (newSettings: Partial<AppSettings>) => {
    const merged = { ...settings, ...newSettings };
    setSettings(merged);
    await saveSettings(merged);

    // Apply to DOM
    document.documentElement.style.setProperty('--glass-blur', `${merged.glassIntensity / 10}px`);
    if (merged.oledMode) {
      document.documentElement.classList.add('oled');
    } else {
      document.documentElement.classList.remove('oled');
    }
  };

  return { settings, updateSettings };
}
```

---

## 5. Micro-Interactions & Animations

Ensure the AI implements these Tailwind classes:
*   **Tap Feedback:** `active:scale-95 transition-transform` on all buttons and cards.
*   **Hover Scaling:** `group-hover:scale-105` on media images.
*   **Glassmorphism:** `backdrop-blur-xl bg-surface-container-high/60`.
*   **Timeline Sticky Headers:** `sticky top-16 z-30`.
*   **Bottom Sheet Modal:** `translate-y-0` (active) and `translate-y-full` (inactive).

