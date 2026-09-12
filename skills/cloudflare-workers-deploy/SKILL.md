---
name: cloudflare-workers-deploy
description: "Deploy a Cloudflare Worker via Wrangler CLI OAuth."
version: 1.0.0
author: Hermes
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [cloudflare, workers, deploy, wrangler, edge, serverless, oauth]
---

# Cloudflare Workers Direct Deploy (Wrangler CLI)

Use when the user has a **Worker project** (a JS/TS/Python entrypoint + `wrangler.jsonc` or `wrangler.toml`) and wants to deploy it to Cloudflare Workers via the **OAuth browser login** flow.

This skill is the counterpart to `cloudflare-pages-deploy`. Where Pages is for **static asset folders**, Workers is for **code that runs at the edge** — JavaScript / Wasm / Python handlers, with optional bindings (KV, R2, D1, Durable Objects, Queues, Workers AI, etc.).

## What this skill does NOT do

- Does not run `npm install` or any build step. The project must already be buildable.
- Does not use GitHub / Git integration. Pure CLI direct upload.
- Does not use API tokens. Login is via `wrangler login` (OAuth, browser opens).
- Does not configure custom domains or DNS. See "Custom domain" section below.
- Does not provision Cloudflare resources (KV namespaces, R2 buckets, D1 databases). See "Bindings" below for prerequisites.

## When to use

- User: "幫我把這個 Worker deploy 到 Cloudflare"
- User: "推這個 edge function 上線"
- User: "deploy 這個後端 API（不是靜態網站）"
- User has any of these project layouts: `wrangler.jsonc` + `src/index.js`, `wrangler.toml` + `worker.js`, etc.
- **User is on SSH / remote shell / headless machine → use `wrangler login --device` (RFC 8628 device flow); user opens the printed URL on any device with a browser to authorize, no localhost access required.**

## When NOT to use

- User wants to deploy a **static site folder** (e.g. `dist/`, `build/`) → use the `cloudflare-pages-deploy` skill instead.
- User wants git push auto-deploy → use Cloudflare dashboard's Git integration, not this skill.
- User wants API-token CI/CD in GitHub Actions → use `cloudflare/wrangler-action`, not this skill.

---

## Workflow

### Step 1 — Verify the Worker project structure

```bash
# Confirm one of the config files exists
test -f wrangler.jsonc && echo "wrangler.jsonc OK" || \
test -f wrangler.toml && echo "wrangler.toml OK" || \
echo "MISSING wrangler config"

# Confirm entrypoint exists (Wrangler defaults to ./src/index.js or ./src/index.ts,
# or whatever `main` in wrangler.jsonc points to)
test -f src/index.js && echo "entry OK" || test -f src/index.ts && echo "entry OK" || \
test -f worker.js && echo "entry OK" || test -f worker.ts && echo "entry OK" || \
echo "MISSING worker entrypoint"
```

If the project has neither a config file nor an entrypoint, **stop** — point the user to `npm create cloudflare@latest -- <name>` or `wrangler init` to scaffold one.

If `node_modules/` is missing, also stop and tell the user to run `npm install`.

### Step 2 — Verify Cloudflare login

```bash
# macOS:    ~/Library/Preferences/.wrangler/config/default.toml
# Linux:    ~/.config/.wrangler/config/default.toml
# Windows:  %APPDATA%\.wrangler\config\default.toml
npx wrangler whoami 2>&1 | head -10
```

- **Logged in** → proceed to Step 3.
- **Not logged in** → run Step 2a.

#### Step 2a — First-time OAuth login

```bash
npx wrangler login
```

Opens the user's default browser to a Cloudflare OAuth page. User clicks "Allow"; CLI receives the token. The browser-based flow needs:
- Port `8976` reachable on localhost (default callback port)
- A desktop browser the user can interact with

**If the user is on a remote / headless / SSH session** (no browser), use the device-flow fallback:
```bash
npx wrangler login --device
```
Wrangler prints a URL + one-time code. User opens it on any device with a browser, enters the code, authorizes. CLI polls for completion. No localhost access required.

Store credentials in OS keychain for better security:
```bash
npx wrangler login --use-keyring
```

### Step 3 — Deploy

```bash
# Default: Wrangler reads wrangler.jsonc / wrangler.toml in the current directory
npx wrangler deploy
```

Wrangler auto-detects the framework (Wrangler 4.x has `--autoconfig` enabled by default), bundles the Worker, uploads it, and prints the live URL.

#### Useful flags

| Flag | Purpose |
|---|---|
| `--name=<name>` | Override the Worker name (default: from `wrangler.jsonc`'s `name` field). |
| `--var KEY=VALUE` | Inject a plaintext environment variable inline (repeatable). |
| `--secrets-file=<path>` | Upload a `.env` or JSON file of secrets additively (does not delete existing secrets). |
| `--assets=<dir>` | Serve a static-assets directory from the same Worker (replaces Workers Sites). |
| `--compatibility-date=<YYYY-MM-DD>` | Pin runtime compatibility (default: Wrangler's latest). |
| `--compatibility-flags=<flag>` | Enable a compat flag (e.g. `nodejs_compat`). |
| `--strict` | Fail the upload if remote has conflicting changes (safer for CI). |
| `--dry-run` | Compile + bundle without uploading (useful for verifying build). |
| `--no-bundle` | Skip internal bundling — only valid for plain JS Workers with no deps. |

Example with all common flags:
```bash
npx wrangler deploy \
  --name=my-api \
  --var LOG_LEVEL=info \
  --secrets-file=.dev.vars \
  --compatibility-date=2026-09-01
```

### Step 4 — Report the URL

Deploy output includes the live URL. Format:
```
<worker-name>.<account-subdomain>.workers.dev
```

Example:
```
✨ Deployment complete! Take a peek over at https://my-api.dchensterebay.workers.dev
```

Save the URL and report it to the user. The Worker is live immediately at the global edge.

---

## Bindings (read `wrangler.jsonc`, don't provision)

Workers gain capability through **bindings** declared in `wrangler.jsonc` / `wrangler.toml`. When `wrangler deploy` runs, it reads the config and:

| Binding | What it does | Resource that must exist before deploy |
|---|---|---|
| `kv_namespaces` | Key-value edge cache | `wrangler kv namespace create <NAME>` |
| `r2_buckets` | Object storage, zero egress | `wrangler r2 bucket create <NAME>` |
| `d1_databases` | Serverless SQL | `wrangler d1 create <NAME>` |
| `queues` | Async message queue | `wrangler queues create <NAME>` |
| `durable_objects` | Stateful singleton per name | declared in config, no CLI create |
| `services` | Call another Worker | target Worker must be deployed |
| `ai` | Workers AI inference | enabled by default |
| `vars` | Plaintext env vars | declared in config |
| `secrets` | Encrypted env vars | `wrangler secret put <KEY>` |

This skill does **not** provision these resources. If the Worker references a binding whose resource doesn't exist, the deploy succeeds but runtime calls will fail. Point the user to the corresponding `wrangler` CLI subcommand if they need to create a resource first.

---

## Verification

After deploy, sanity-check the live URL:

```bash
URL="https://<worker-name>.<account-subdomain>.workers.dev"
curl -sIL "$URL" | head -5            # check HTTP 200
curl -s "$URL" | head -20              # check response body
```

For first deploy to a fresh `<name>.workers.dev` subdomain, Cloudflare may briefly return **523 errors** while DNS propagates. Wait ~1 minute and retry.

---

## Custom domain (handoff, not part of this skill)

This skill deploys to `<worker-name>.<account-subdomain>.workers.dev`. To bind a real domain:

1. In Cloudflare dashboard → Workers & Pages → select Worker → **Settings** → **Triggers** → **Custom Domains** → **Add Custom Domain**
2. Enter the domain; if it is on Cloudflare, DNS is auto-configured.
3. If the domain is elsewhere, point a CNAME to `<worker-name>.<account-subdomain>.workers.dev`.

Or in `wrangler.jsonc`:
```jsonc
{
  "routes": [{ "pattern": "api.example.com/*", "zone_name": "example.com" }]
}
```

Tell the user this is dashboard-or-config-only; this skill does not edit DNS.

---

## Common pitfalls

### Missing or unreadable `wrangler.jsonc`
- If the config file has a syntax error or refers to a missing entrypoint, `wrangler deploy` fails fast with a parse error. Always `test -f wrangler.jsonc` first.
- If `main` points to a file that doesn't exist, deploy fails. Either fix the path or run `wrangler init` to regenerate.

### `node_modules` missing
- `wrangler deploy` will fail if `package.json` lists dependencies that aren't installed. Run `npm install` first.

### Vars vs Secrets confusion
- **Vars** (`vars` in config / `--var` flag): plaintext, visible in dashboard, included in deploy bundle.
- **Secrets** (`wrangler secret put KEY` / `--secrets-file`): encrypted, never visible, not in bundle.
- Never put API keys / tokens in `vars` — always use secrets.

### First-deploy 523 errors
- Fresh `*.workers.dev` subdomains propagate DNS within ~1 minute. Wait and retry.
- Persistent 523s usually mean the Worker is throwing — check `wrangler tail <name>` for live logs.

### Compatibility date too old
- If `wrangler.jsonc` pins `compatibility_date` to a date from >1 year ago, newer runtime APIs are unavailable. Use `--latest` or update the date.

### Worker name conflict
- `<name>.workers.dev` is unique per account. If `my-api` is already used, change the `name` field in `wrangler.jsonc`.

### Bindings reference non-existent resources
- Deploy succeeds; runtime fails. Verify resources exist via `wrangler kv namespace list`, `wrangler r2 bucket list`, `wrangler d1 list`, etc.

### `--strict` in CI
- Use `--strict` to prevent accidental remote overwrite when multiple deployments run concurrently. Without it, the latest deploy wins silently.

---

## Quick reference (TL;DR)

```bash
# 1. One-time login (OAuth, browser opens)
npx wrangler login

# 2. Deploy (from the Worker project directory)
npx wrangler deploy

# → Output: https://<worker-name>.<account-subdomain>.workers.dev
```

That's it. No GitHub, no API token, no CI/CD config. Reads `wrangler.jsonc` / `wrangler.toml` for bindings.