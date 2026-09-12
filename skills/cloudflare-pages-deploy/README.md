# `cloudflare-pages-deploy`

A [Hermes Agent](https://hermes-agent.nousresearch.com/) skill that deploys a pre-built static site folder to [Cloudflare Pages](https://pages.cloudflare.com/) using the [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) OAuth login flow.

See the [parent README](../../README.md) for installation, requirements, and platform notes.

## Trigger

Use when the user has a **pre-built static site folder** and wants to push it:

- `dist/` (Vite / Astro / Nuxt / SvelteKit)
- `build/` (Gatsby / Create React App)
- `out/` (Next.js static export)
- `_site/` (Eleventy / Hugo)
- `public/` (plain static)

## TL;DR

```bash
# One-time login (OAuth, browser opens)
npx wrangler login

# Deploy
npx wrangler pages deploy dist --project-name=my-site

# → https://main.my-site.pages.dev
```

For SSH / headless machines: `npx wrangler login --device` instead of `wrangler login`.

## Source

The actual skill definition is [`SKILL.md`](./SKILL.md). Install by copying it into your Hermes skills directory:

```bash
mkdir -p ~/.hermes/skills/devops/cloudflare-pages-deploy
cp SKILL.md ~/.hermes/skills/devops/cloudflare-pages-deploy/
```
