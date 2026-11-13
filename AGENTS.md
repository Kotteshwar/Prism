# AGENTS.md

**Project:** Prism Gallery — a single-user PWA media gallery backed by Koofr WebDAV.  
**Stack:** React 18 + TypeScript + Vite + Tailwind + Cloudflare Pages Functions + Workers KV + `idb-keyval`.

This file defines rules for any AI coding agent working on this repository. Read it fully before editing code. Read `docs/SKILL.md` and the six design documents in `docs/` for detailed contracts.

---

## 1. Read Before You Write

Before writing code, read these files in order:

1. `docs/SKILL.md` — master context, conventions, design tokens.
2. `docs/01-PRD.md` — what to build and why.
3. `docs/02-ARCHITECTURE.md` — system structure, proxy rules, free-tier limits.
4. `docs/03-DATA-MODEL.md` — TypeScript interfaces and WebDAV contracts.
5. `docs/04-UI-SPEC.md` — screen-to-component mapping.

If a rule in this file conflicts with a design document, the **design document wins**. Then fix this file.

---

## 2. Code Quality Rules

### 2.1 Write simple, human-readable code

- Prefer clear names over short names. `fileInfoPanel`, not `fip`.
- Prefer early returns over nested `if` blocks.
- Prefer explicit values over magic numbers. Extract constants at the top of the file.
- One function does one thing. If a function needs a comment to explain what it does, split it.
- Delete unused code before committing. Do not leave commented-out lines.

### 2.2 No repeated functions

- Before writing a helper, search `src/lib/` and `src/components/shared/` for an existing one.
- If two files contain the same logic, extract it into `src/lib/` and import it in both.
- If a React component appears in two places, move it to `src/components/shared/`.

### 2.3 File size limits

- **Component files:** 200 lines max. Split into subcomponents if longer.
- **Hook files:** 100 lines max. Split by concern.
- **Utility files (`src/lib/`):** 150 lines max. Group by domain.

### 2.4 No unwanted lines

- No `console.log` in committed code. Use `console.error` only for caught errors that the user cannot see.
- No `TODO` comments without a GitHub issue link.
- No dead imports.
- No empty `useEffect` or unused state.
- No `any` in TypeScript unless a third-party library forces it. If so, add a one-line comment explaining why.

---

## 3. Shared Components Rule

Every reusable UI element lives in `src/components/shared/`. Never duplicate visual patterns.

### 3.1 Mandatory shared components

Build these **once** and reuse everywhere:

| Component | Used in | Purpose |
|---|---|---|
| `BottomSheet` | Create Album, Share Sheet, User Editor | Slide-up modal with backdrop and drag handle |
| `Modal` | Delete confirmation, folder picker | Centered dialog |
| `GlassHeader` | Every screen | Sticky top header with `bg-surface/70 backdrop-blur-xl` |
| `BottomNav` | Every authenticated screen | Floating pill nav with 3 tabs |
| `MediaCell` | Photos grid, Videos grid, Collections | Renders an image or video thumbnail with overlays |
| `EmptyState` | Photos, Videos, Collections, Admin users | Icon + message + optional action |
| `PrimaryButton` | Every form | The `bg-primary text-on-primary` button |
| `DangerButton` | Delete flows | The `text-error` button variant |
| `IconButton` | Header actions | Round icon-only button with `active:scale-95` |
| `ToggleSwitch` | Settings, Share Sheet | The Tailwind peer-checked switch |
| `FormField` | Login, Create Album, User Editor | Label + input + error text |
| `Toast` | Everywhere | Bottom notification for success/error |

If a new screen needs something that looks similar to a shared component, extend the shared component with props. Do not copy-paste.

### 3.2 Component structure

Every shared component follows this layout:

```tsx
interface Props {
  // explicit, typed props
}

export function ComponentName({ ... }: Props) {
  // 1. hooks
  // 2. derived values
  // 3. handlers
  // 4. early returns for loading/error/empty
  // 5. JSX
}
```

No default exports. Named exports only.

---

## 4. File Structure

Follow this structure exactly. Do not invent new top-level folders.

```text
src/
  components/
    shared/          # BottomSheet, Modal, MediaCell, GlassHeader, BottomNav, ...
    photos/          # Photos-specific (Timeline, DateCluster)
    videos/          # Videos-specific (VideoGrid, VideoPlayer)
    collections/     # Collections-specific (AlbumCard, AlbumCollage)
    admin/           # AdminUserList, UserEditorModal, AlbumAccessPicker
  pages/             # One file per route
  hooks/             # One concern per hook
  lib/               # Pure logic: webdav.ts, media.ts, localState.ts, session.ts
  types/             # Shared TypeScript types
functions/           # Cloudflare Pages Functions (file-based routing)
docs/                # Design documents (read-only for agents)
```

---

## 5. Cloudflare Functions Rules

- Every function uses `export async function onRequest(context)`.
- Never import Node.js modules (`fs`, `path`, `crypto` from Node). Use Web APIs (`crypto.subtle`).
- The proxy (`functions/api/dav/[[path]].ts`) only forwards methods in the allowlist: `PROPFIND`, `GET`, `PUT`, `MKCOL`, `DELETE`, `MOVE`.
- Reject any path containing `..` before forwarding.
- Stream `PUT` request bodies: `fetch(url, { body: request.body })`. Never read into a buffer.
- Every admin route (`/api/admin/*`) must call `requireAdmin(context)` as its first line.
- Never log credentials, session tokens, or share tokens.

---

## 6. Client-Side Rules

- Fetch media metadata with `Range: bytes=0-1048575` only. Never download full media for metadata.
- Use `exifreader` for images and `mp4box.js` for videos. Do not add another parsing library.
- Use `idb-keyval` for local state. Do not add `Dexie`, `localforage`, or `redux-persist`.
- Merge local state (`isFavorite`, `isHidden`) into `MediaItem` after fetching from WebDAV. Never store local flags on the server.
- Use `navigator.share` for OS-level sharing. Do not build custom share dialogs for external apps.

---

## 7. UI Rules

- Use only the Tailwind tokens defined in `tailwind.config.js`. No arbitrary values like `bg-[#123456]`.
- Use only the fonts `font-headline`, `font-body`, `font-label`. No other font utilities.
- Use only the shared components listed in §3.1. Do not create one-off buttons, headers, or modals.
- Every interactive element has `active:scale-95 transition-transform`.
- Every glass surface uses `backdrop-blur-xl bg-surface-container/80`.
- Every screen respects safe areas: `pt-safe` on headers, `pb-safe` on bottom elements.

---

## 8. Text Rules (ASD-STE 100)

- Procedural sentences: 20 words max. Descriptive sentences: 25 words max.
- One instruction per sentence. One topic per paragraph.
- Active voice. No contractions. No semicolons.
- Use approved vocabulary: "log in" (not sign in), "delete" (not remove), "album" (not folder).
- Sentence case for headings and labels.
- State the consequence before the action in warnings.
- Reference UI labels exactly: if the button says **Delete**, write "Tap **Delete**."

Full rules in `docs/ASD-STE-100.md`.

---

## 9. TypeScript Rules

- No `any`. Use `unknown` and narrow with type guards.
- Every `MediaItem`, `MediaMetadata`, `ViewerUser`, `ShareRecord`, `AppSettings` interface lives in `src/types/`. Do not redeclare them.
- Every API response shape has a matching TypeScript interface.
- Every `catch` block either re-throws, returns a typed error, or sets a user-visible error state. Silent catches are forbidden.

---

## 10. Testing Rules

- Write a test before implementing a pure function in `src/lib/`. Use Vitest.
- Write a test for every parser: `parsePropfindXml`, `filenameDate`, `extractImageMetadata`, `extractVideoMetadata`, `buildRenameTarget`.
- Do not test React components unless they contain conditional logic. Test the hooks and lib functions they call.
- Every test file lives next to its source file: `webdav.ts` → `webdav.test.ts`.

---

## 11. Before You Commit

Run these checks. Do not commit if any fail.

- [ ] `npm run typecheck` passes.
- [ ] `npm run lint` passes.
- [ ] `npm run test` passes.
- [ ] No `console.log` remains.
- [ ] No unused imports.
- [ ] No new duplicate components (searched `src/components/shared/`).
- [ ] No new duplicate helper functions (searched `src/lib/`).
- [ ] No file exceeds the size limits in §2.3.
- [ ] No secrets in the diff (searched for `KOOFR_APP_PASSWORD`, `ADMIN_PASSPHRASE`, `SESSION_SECRET`).
- [ ] Every new UI string follows §8.

---

## 12. When in Doubt

- If a rule in this file is unclear, ask the human before proceeding.
- If a design document contradicts this file, follow the design document and flag the conflict.
- If a task requires adding a new shared component, add it to §3.1 in the same commit.
- If a task requires a new file structure, do not create it without the human's approval.