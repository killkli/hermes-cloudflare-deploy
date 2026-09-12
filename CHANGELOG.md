# Changelog

All notable changes to this skill collection are documented here. The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.0] — 2026-09-12

### Added
- Monorepo consolidating the two Cloudflare deploy skills for [Hermes Agent](https://hermes-agent.nousresearch.com/).
- `cloudflare-pages-deploy` v1.0.0 — deploy a pre-built static site folder via `wrangler pages deploy` + OAuth.
- `cloudflare-workers-deploy` v1.0.0 — deploy a Worker project via `wrangler deploy` + OAuth, with bindings (KV / R2 / D1 / Durable Objects / Queues / Workers AI) read from `wrangler.jsonc` / `wrangler.toml`.
- Shared OAuth flow: `wrangler login` (browser) and `wrangler login --device` (RFC 8628 device flow for SSH / headless).
- Cross-platform credential-path notes (macOS `~/Library/Preferences`, Linux `~/.config`, Windows `%APPDATA%`).

### Notes
- Both skills are **CLI direct deploy** — no GitHub integration, no API tokens, no CI/CD config.
- `cloudflare-workers-deploy` does not provision bindings — the user creates KV namespaces / R2 buckets / D1 databases via `wrangler` CLI subcommands separately.
- `cloudflare-workers-deploy` reflects Cloudflare's 2026 guidance to prefer Workers over Pages for new full-stack projects (Workers gained static-asset serving via `--assets`).
