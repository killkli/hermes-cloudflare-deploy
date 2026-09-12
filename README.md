# Hermes Cloudflare Deploy Skills

A monorepo of [Hermes Agent](https://hermes-agent.nousresearch.com/) skills for deploying projects to [Cloudflare](https://cloudflare.com/) using the [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) OAuth login flow.

**No GitHub. No API tokens. No CI/CD config.** One browser-based login, then a single CLI deploy per skill.

## Skills in this repo

| Skill | Use when | Deploys to |
|---|---|---|
| [`cloudflare-pages-deploy`](./skills/cloudflare-pages-deploy/) | You have a **pre-built static site folder** (`dist/`, `build/`, `public/`, `out/`, `_site/`) | `<project>.pages.dev` |
| [`cloudflare-workers-deploy`](./skills/cloudflare-workers-deploy/) | You have a **Worker project** (JS/TS/Python entrypoint + `wrangler.jsonc` / `wrangler.toml`) | `<worker-name>.<account-subdomain>.workers.dev` |

Both skills:

- Use **OAuth browser login** (`wrangler login`) — no API tokens, no service accounts.
- Support **device-flow login** for SSH / remote / headless machines (`wrangler login --device`, RFC 8628).
- Use the **same Wrangler CLI** — you only need `npx` to fetch it; no global install.
- Require only a **free Cloudflare account** (no credit card).

## Choosing between Pages and Workers

Cloudflare's 2026 recommendation is to default to **Workers** for new full-stack projects (since Workers gained static-asset serving via `--assets`). Pages remains supported for static-only sites.

| Scenario | Use |
|---|---|
| Pure static site (HTML / CSS / JS, no backend) | **Pages** |
| Static site + simple API (single deploy, no DB) | **Workers** with `--assets` |
| API endpoints, SSR, edge functions | **Workers** |
| Anything needing KV / R2 / D1 / Durable Objects / Queues / Workers AI | **Workers** (with bindings in `wrangler.jsonc`) |
| Cron jobs, scheduled tasks | **Workers** (with `triggers` in `wrangler.jsonc`) |

If unsure, start with Workers — it covers everything Pages can do plus much more.

## Installation

Each skill is a single `SKILL.md` file. Copy it into your Hermes skills directory:

```bash
# Personal profile (default)
mkdir -p ~/.hermes/skills/devops/cloudflare-pages-deploy
cp skills/cloudflare-pages-deploy/SKILL.md ~/.hermes/skills/devops/cloudflare-pages-deploy/

mkdir -p ~/.hermes/skills/devops/cloudflare-workers-deploy
cp skills/cloudflare-workers-deploy/SKILL.md ~/.hermes/skills/devops/cloudflare-workers-deploy/
```

For other Hermes profiles:

```bash
# ~/.hermes/profiles/<name>/skills/devops/...
```

The skills trigger automatically on deploy-related prompts — no extra setup needed.

## Requirements

- Node.js ≥ 18 (for `npx`)
- A [free Cloudflare account](https://dash.cloudflare.com/sign-up) (no credit card needed)
- Wrangler is fetched on-demand via `npx` — no global install

## Platform notes

### macOS
Wrangler stores OAuth credentials at `~/Library/Preferences/.wrangler/config/default.toml`. Optionally store in OS keychain: `npx wrangler login --use-keyring`.

### Linux
Credentials at `~/.config/.wrangler/config/default.toml`.

### Windows
Credentials at `%APPDATA%\.wrangler\config\default.toml`.

### Headless / SSH / remote machines
```bash
npx wrangler login --device
```
Wrangler prints a URL + one-time code. Open the URL on any device with a browser, enter the code, authorize. The CLI polls for completion. **No localhost browser access required.**

## License

MIT — see [LICENSE](./LICENSE).
