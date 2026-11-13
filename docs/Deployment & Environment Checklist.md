# Deployment & Environment Checklist: Prism Gallery

**Version:** 1.0  
**Depends on:** All prior documents (PRD v2.0 → Build Plan v2.0)  
**Target:** Cloudflare Pages (Free Plan) + Pages Functions + Workers KV + Koofr WebDAV

---

## 1. Prerequisites

Before starting, confirm you have:

| Requirement | Notes |
|---|---|
| Koofr account | Active, with WebDAV access enabled (default on paid/free accounts) |
| Cloudflare account | Free plan is sufficient |
| Node.js ≥ 18 | For Wrangler CLI and Vite |
| Wrangler CLI | `npm install -g wrangler` or use `npx wrangler` |
| Git repository | GitHub/GitLab repo connected to Cloudflare Pages (or use direct upload) |

---

## 2. Koofr Credentials Setup

### 2.1 Generate the WebDAV App Password

Koofr does **not** use your login password for WebDAV. You must generate an application-specific password.

1. Log in to the Koofr web app.
2. Click your profile picture (top-right) → **Preferences**.
3. Select **Password** from the left menu.
4. Scroll to **App passwords**.
5. Type a name (e.g., `Prism Gallery`) in the **Generate new password** field.
6. Click **Generate**.
7. **Copy the password immediately.** Koofr will never display it again.

### 2.2 Record the Credentials

You will need two values:

| Value | Where to find it | Example |
|---|---|---|
| `KOOFR_USER` | Your Koofr account email | `you@example.com` |
| `KOOFR_APP_PASSWORD` | The generated app password | `abcd1234-efgh-5678-ijkl-9012mnop3456` |

Store these in a password manager. Do **not** commit them to the repository.

---

## 3. Cloudflare Project Setup

### 3.1 Create the Pages Project

**Option A — Git-connected (recommended):**

1. Push the project to GitHub.
2. In the Cloudflare Dashboard → **Workers & Pages** → **Create** → **Pages** → **Connect to Git**.
3. Select the repository.
4. Set build settings:
   - **Framework preset:** Vite
   - **Build command:** `npm run build`
   - **Build output directory:** `dist`
5. Click **Save and Deploy** (first deployment can happen before secrets are set; Functions will fail at runtime until secrets are configured).

**Option B — Direct upload:**

```bash
npm run build
npx wrangler pages deploy dist --project-name=prism-gallery
```

### 3.2 Create the KV Namespace

Run:

```bash
npx wrangler kv namespace create SHARE_KV
```

The output returns a namespace ID (e.g., `116847d7b0714a9c8d2882335b05d35a`). Copy it.

### 3.3 Bind the KV Namespace

**Via `wrangler.toml` (recommended):**

```toml
name = "prism-gallery"
compatibility_date = "2026-11-01"
pages_build_output_dir = "dist"

[[kv_namespaces]]
binding = "SHARE_KV"
id = "<paste-namespace-id-here>"

[vars]
APP_ORIGIN = "https://prism-gallery.pages.dev"
KOOFR_USER = "you@example.com"
# KOOFR_APP_PASSWORD, ADMIN_PASSPHRASE, SESSION_SECRET set via wrangler secret put
```

**Via Cloudflare Dashboard (alternative):**

1. Go to your Pages project → **Settings** → **Bindings** → **Add** → **KV namespace**.
2. Set **Variable name** to `SHARE_KV`.
3. Select the namespace you created.
4. Redeploy the project for the binding to take effect.

### 3.4 Set Secrets

Secrets are encrypted and never exposed in the repository or build output. Set them via Wrangler:

```bash
npx wrangler pages secret put KOOFR_APP_PASSWORD --project-name=prism-gallery
npx wrangler pages secret put ADMIN_PASSPHRASE --project-name=prism-gallery
npx wrangler pages secret put SESSION_SECRET --project-name=prism-gallery
```

For each command, Wrangler prompts you to paste the value securely.

| Secret | Value source | Notes |
|---|---|---|
| `KOOFR_APP_PASSWORD` | Koofr app password (§2.1) | Never the login password |
| `ADMIN_PASSPHRASE` | Your chosen admin passphrase | Used for the login screen |
| `SESSION_SECRET` | A random 32+ byte string | Generate with `openssl rand -base64 32` |

**Important:** Secrets are bound at deploy time. After setting or changing a secret, trigger a new deployment for the running build to see it.

Verify all three exist:

```bash
npx wrangler pages secret list --project-name=prism-gallery
```

---

## 4. Local Development Environment

### 4.1 Local Secrets

For `wrangler pages dev`, create a `.dev.vars` file in the project root (add to `.gitignore`):

```ini
KOOFR_USER=you@example.com
KOOFR_APP_PASSWORD=your-app-password
ADMIN_PASSPHRASE=local-dev-passphrase
SESSION_SECRET=local-dev-secret-at-least-32-bytes-long
```

### 4.2 Run Locally with KV Binding

```bash
npm run build
npx wrangler pages dev dist --kv=SHARE_KV
```

The `--kv=SHARE_KV` flag binds a local KV emulation to the `SHARE_KV` variable name. If the binding is also declared in `wrangler.toml`, the command-line flag takes precedence.

### 4.3 Local Verification Checklist

| Check | Command / Action | Expected |
|---|---|---|
| Auth works | `curl -X POST http://localhost:8788/api/auth -d '{"passphrase":"..."}'` | `Set-Cookie` header |
| Proxy works | Browser fetch to `/api/dav/` with cookie | `207 Multi-Status` XML |
| KV works | `POST /api/share` with valid paths | Returns `{ url, token }` |
| Range probe | `GET /api/dav/{path}` with `Range: bytes=0-1048575` | `206 Partial Content` |

---

## 5. Production Deployment

### 5.1 Deploy

If Git-connected, push to the connected branch. Otherwise:

```bash
npm run build
npx wrangler pages deploy dist --project-name=prism-gallery
```

### 5.2 Post-Deploy Smoke Tests

Run these against the production URL:

1. **Login:** Open `https://prism-gallery.pages.dev/login`, enter `ADMIN_PASSPHRASE`. Confirm redirect to `/photos`.
2. **Gallery loads:** Confirm photos appear grouped by month. If empty, check that `/Photos` exists in Koofr and that the proxy returns `207`.
3. **Metadata probe:** Open a photo's info panel. Confirm EXIF fields populate within ~1.5s.
4. **Share link:** Create a share, open the `/share/:token` URL in an incognito window. Confirm media renders without a login cookie.
5. **Share expiry:** Create a share with the minimum TTL (60s). Wait 2 minutes, reload. Confirm `404`.
6. **PWA install:** On mobile, use "Add to Home Screen". Confirm standalone launch.

---

## 6. Cloudflare Free Tier Limits

All operations must stay within these limits. At 1–3 occasional users, usage will be far below them.

| Resource | Free Limit | Prism Usage Estimate |
|---|---|---|
| Pages Functions requests | 100,000 / day | < 500 / day |
| Functions CPU time | 10 ms / request | ~1–3 ms (thin proxy) |
| KV reads | 100,000 / day | < 100 / day (share views only) |
| KV writes | 1,000 / day | < 10 / day (share creation) |
| KV stored data | 1 GB | < 1 KB (tokens only) |
| Static asset bandwidth | Unlimited | Unlimited |

**KV billing note:** Fetches for non-existent keys (e.g., expired share tokens) still count as reads. This is negligible at your scale but worth knowing.

---

## 7. Security Checklist

Before considering the deployment "done":

- [ ] `KOOFR_APP_PASSWORD` exists only as a Cloudflare secret (never in `wrangler.toml`, `.env` committed to Git, or client bundle).
- [ ] `SESSION_SECRET` is ≥ 32 random bytes.
- [ ] The session cookie is `HttpOnly`, `Secure`, `SameSite=Lax`.
- [ ] The proxy rejects paths containing `..` (directory traversal).
- [ ] The proxy only allows `PROPFIND`, `GET`, `PUT`, `MKCOL`, `DELETE`, `MOVE` (all other methods return `405`).
- [ ] Share tokens are ≥ 256 bits of entropy (32 random bytes, base64url-encoded).
- [ ] Share viewer validates exact path match against KV whitelist; unlisted paths return `403`.
- [ ] The Service Worker's `navigateFallbackDenylist` includes `/api/` and `/share/` (prevents HTML shell from being served to media requests).

---

## 8. Troubleshooting Quick Reference

| Symptom | Likely cause | Fix |
|---|---|---|
| `401` on all `/api/dav/*` | Missing or expired cookie | Re-login; check `SESSION_SECRET` consistency |
| `207` but empty gallery | Koofr folder path wrong, or `PROPFIND` depth not 1 | Verify `/Photos` exists; check request headers |
| `403` on valid share link | Path not in `ShareRecord.paths` (exact match required) | Re-create the share with the correct paths |
| `500` on `/api/share` | KV binding missing or namespace ID wrong | Check `wrangler.toml` binding and redeploy |
| Secrets not visible after `secret put` | Deployment not refreshed | Trigger a new deploy |
| EXIF panel stays empty | Range request blocked or `ExifReader` threw | Check browser console; verify `206` response |
| MP4 metadata times out | `moov` box not in first 1 MiB | Expected for some files; filename date is shown instead |

---

## 9. Final Deliverables Summary

You now have a complete six-document package:

| Doc | Title | Purpose |
|---|---|---|
| 1 | PRD v2.0 | What to build and why |
| 2 | Technical Architecture v2.0 | How the system is structured |
| 3 | Data Model & WebDAV Spec v2.0 | Exact API contracts and parsers |
| 4 | UI/UX Component Spec v2.0 | Screen-to-component mapping |
| 5 | Vibe-Coding Build Plan v2.0 | Step-by-step AI prompts with acceptance gates |
| 6 | Deployment & Environment Checklist | External services, secrets, and launch verification |
