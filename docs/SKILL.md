---
name: prism-gallery
description: Master context for building the Prism Gallery PWA. Use this for all React, Cloudflare, WebDAV, and EXIF tasks.
---

# Prism Gallery Developer Context

You are an expert React and Cloudflare engineer building "Prism", a dark-mode media gallery PWA for Koofr WebDAV.

Before writing any code, you MUST adhere to the following rules:

## 1. Core Architecture
- **Backend:** Cloudflare Pages Functions. Use `export async function onRequest(context)`.
- **Proxy:** Do NOT use Node.js built-ins. Stream `PUT` requests directly to `fetch()`. Limit CPU to 10ms.
- **WebDAV Methods:** Only allow `PROPFIND`, `GET`, `PUT`, `MKCOL`, `DELETE`, `MOVE`.
- **CORS:** Koofr does not support CORS. All Koofr requests MUST go through `/api/dav/*`.

## 2. Media Metadata Rules
- **Images:** Use `exifreader` on the client side. 
- **Videos:** Use `mp4box.js` on the client side. 
- **Probing:** Fetch metadata using a `Range: bytes=0-1048575` (1 MiB) header. NEVER download the full file for metadata.
- **Fallback:** If metadata is missing, parse `YYYYMMDD_HHMMSS` from the filename.

## 3. Local State
- Use `idb-keyval` for Favorites, Hidden files, and Settings (Glass intensity, OLED mode).
- Do NOT use Redux or Zustand. Use React Context and custom hooks.

## 4. Design System (Strict)
- **Colors:** Use `bg-surface`, `bg-surface-container-lowest`, `text-primary`, `text-tertiary` (defined in tailwind.config.js).
- **Typography:** Use `font-headline` (Epilogue), `font-body` (Inter), `font-label` (Google Sans).
- **Aesthetics:** Use heavy `backdrop-blur-xl` and `bg-surface-container/80` for glassmorphism.

## 5. Text & Copywriting
- Follow ASD-STE 100 rules.
- Active voice. Max 20 words per sentence. One instruction per sentence.
- UI Labels: "Log in" (not Sign in), "Delete" (not Remove), "Album" (not Folder).

## Reference Documents
For detailed schemas and API contracts, refer to:
- `PRD.md` (Doc 1)
- `ARCHITECTURE.md` (Doc 2)
- `DATA_MODEL.md` (Doc 3)
- `UI_SPEC.md` (Doc 4)