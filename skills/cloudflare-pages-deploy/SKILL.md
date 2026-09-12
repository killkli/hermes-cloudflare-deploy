---
name: cloudflare-pages-deploy
description: "Deploy built folder to CF Pages via Wrangler OAuth."
version: 1.0.0
author: Hermes
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [cloudflare, pages, deploy, static-site, wrangler, hosting, cdn]
---

# Cloudflare Pages Direct Deploy (Wrangler CLI)

Use when the user has a **pre-built** static site folder (e.g. `dist/`, `build/`, `public/`, `_site/`, `out/`) and wants to push it to Cloudflare Pages via the **OAuth browser login** flow.

## What this skill does NOT do

- Does not run the build step. The folder must already contain the final static files.
- Does not use GitHub / Git integration. Pure CLI direct upload.
- Does not use API tokens. Login is via `wrangler login` (OAuth, browser opens).
- Does not configure custom domains or DNS. See "Custom domain" section below for handoff.

## When to use

- User: "幫我把這個網站 deploy 到 Cloudflare Pages"
- User: "dist 已經 build 好，上線到 pages"
- User: "推到 Cloudflare Pages，不要用 GitHub"
- User has any of these folders ready: `dist`, `build`, `out`, `public`, `_site`
- **User is on SSH / remote shell / headless machine → use `wrangler login --device` (RFC 8628 device flow); user opens the printed URL on any device with a browser to authorize, no localhost access required.**

## When NOT to use

- User wants git push auto-deploy → use Cloudflare dashboard's Git integration, not this skill
- User has SSR / dynamic features → this skill is for **static** assets only (Pages Functions are a separate concern)
- User wants API-token CI/CD in GitHub Actions → use `cloudflare/wrangler-action`, not this skill

---

## Workflow

### Step 1 — Verify the build folder exists

```bash
# Common build output paths. Confirm one exists.
ls -la dist/ 2>/dev/null || ls -la build/ 2>/dev/null || ls -la public/ 2>/dev/null
```

If multiple exist, ask the user which one to deploy. If none exist, **stop** — do not deploy an empty or wrong folder.

Sanity-check the folder has an `index.html`:
```bash
test -f dist/index.html && echo "OK" || echo "MISSING index.html — not a static site?"
```

### Step 2 — Verify Cloudflare login

```bash
# macOS: ~/Library/Preferences/.wrangler/config/default.toml
# Linux: ~/.config/.wrangler/config/default.toml
# Windows: %APPDATA%\.wrangler\config\default.toml
npx wrangler whoami 2>&1 | head -10
```

- **Logged in** → proceed to Step 3.
- **Not logged in** → run Step 2a.

#### Step 2a — First-time OAuth login

```bash
npx wrangler login
```

This opens the user's default browser to a Cloudflare OAuth page. The user clicks "Allow", then the CLI receives the token. The browser-based flow needs:
- Port `8976` reachable on localhost (default callback port)
- A desktop browser the user can interact with

**If the user is on a remote / headless / SSH session** (no browser), use the device-flow fallback:
```bash
npx wrangler login --device
```
This prints a URL + code; user opens it on any device to authorize. More friction but works without localhost access.

After login, store credentials in OS keychain (more secure than plaintext config):
```bash
# Re-login with keyring flag (overwrites existing config)
npx wrangler login --use-keyring
```

### Step 3 — Deploy

```bash
# Default: deploy dist/ to a project (auto-created on first deploy)
npx wrangler pages deploy dist --project-name=<name>
```

- `<name>` becomes the `<name>.pages.dev` URL.
- **Wrangler 4.x does NOT auto-create the Pages project on first deploy.** If the project doesn't exist, deploy will fail with `The Pages project "<name>" does not exist`. Create it first:

  ```bash
  npx wrangler pages project create <name> --production-branch=main
  npx wrangler pages deploy dist --project-name=<name>
  ```

  (Wrangler 3.x auto-created on first deploy; this changed in 4.x and is a common gotcha.)
- Subsequent deploys with the same `--project-name` update the existing project.

#### Useful flags

| Flag | Purpose |
|---|---|
| `--project-name=<name>` | Target project. **Required.** Must exist (create first via `wrangler pages project create` — see Step 3). |
| `--branch=<branch>` | Branch label for this deployment (default: `main`). Shows in dashboard. |
| `--commit-hash=<sha>` | Tag deployment with a commit SHA. Visible in dashboard history. |
| `--commit-message="..."` | Tag with a message. |
| `--commit-dirty=true` | Mark workspace dirty (useful when deploying without git). |

Example with all tags:
```bash
npx wrangler pages deploy dist \
  --project-name=my-site \
  --branch=production \
  --commit-message="v1.0 release"
```

### Step 4 — Report the URL

The deploy output includes the preview URL. Format:
```
<commit-or-branch>.<project-name>.pages.dev
```

Example real output:
```
✨ Deployment complete! Take a ⬇️  to keep going...
🌎 Deploying to your site...  https://main.my-site.pages.dev
```

Save the URL and report it to the user. The site is live immediately.

---

## Verification

After deploy, sanity-check the live URL:

```bash
URL="https://<branch>.<project-name>.pages.dev"
curl -sIL "$URL" | head -5   # check HTTP 200
curl -s "$URL" | grep -i "<title>"   # check the page title made it
```

Or open in browser via the user's default browser:
```bash
open "https://<branch>.<project-name>.pages.dev"   # macOS
xdg-open "https://..."                              # Linux
```

---

## Custom domain (handoff, not part of this skill)

This skill deploys to `<project>.pages.dev`. To bind a real domain:

1. In Cloudflare dashboard → Workers & Pages → select project → **Settings** → **Custom domains** → **Set up a custom domain**
2. Enter the domain; if it is on Cloudflare, DNS is auto-configured.
3. If the domain is elsewhere, point a CNAME to `<project>.pages.dev`.

Tell the user this step is dashboard-only and link them to it; do not attempt DNS edits from CLI.

---

## Common pitfalls

### Wrangler 4.x does not auto-create Pages projects
- First-time deploy against a fresh project name fails with `The Pages project "<name>" does not exist`. (Wrangler 3.x auto-created.)
- Fix: run `npx wrangler pages project create <name> --production-branch=main` once, then redeploy.
- The hint text suggesting you should use Workers instead is misleading — direct Pages deploy still works fine, you just have to create the project first.

### Empty / wrong folder deployed
- Always `ls` the target folder first. A missing or stale build silently uploads a half-broken site.
- If the user runs `npm run build` themselves, ask for the **output folder name** explicitly — frameworks differ (`dist` for Vite/Astro/Nuxt/SvelteKit, `build` for Gatsby/CRA, `out` for Next static export, `_site` for Eleventy/Hugo, `public` for plain static).

### Wrangler not installed globally
- The skill uses `npx wrangler` (fetches latest), so no global install needed.
- First run takes ~10s to download the wrangler package. Subsequent runs are fast.

### "Authentication error [code: 10000]"
- OAuth token expired or invalid. Re-run `wrangler login`.

### User on a headless / remote machine
- Default `wrangler login` opens localhost:8976. If the user's browser can't reach the host (e.g. SSH session without port forwarding), switch to `wrangler login --device` (RFC 8628 device flow).

### Project name collision
- `<project>.pages.dev` is globally unique on Cloudflare. If `my-site` is taken, pick another or use `--project-name` to choose a unique slug.

### Large folders (slow upload)
- Pages direct upload has a per-file and total-size limit (check current docs). For very large media-heavy sites, prefer the Git integration so Cloudflare builds in parallel.

---

## Quick reference (TL;DR)

```bash
# 1. One-time login (OAuth, browser opens)
npx wrangler login

# 2. Deploy
npx wrangler pages deploy dist --project-name=my-site

# Output gives you: https://main.my-site.pages.dev
```

That's it. No GitHub, no API token, no CI/CD config.