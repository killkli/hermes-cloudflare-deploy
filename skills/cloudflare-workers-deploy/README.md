# `cloudflare-workers-deploy`

A [Hermes Agent](https://hermes-agent.nousresearch.com/) skill that deploys a [Cloudflare Worker](https://developers.cloudflare.com/workers/) project using the [Wrangler CLI](https://developers.cloudflare.com/workers/wrangler/) OAuth login flow.

See the [parent README](../../README.md) for installation, requirements, and platform notes.

## Trigger

Use when the user has a **Worker project** (JS / TS / Python entrypoint + `wrangler.jsonc` or `wrangler.toml`) and wants to deploy it to the Cloudflare edge.

Capabilities unlocked through bindings declared in `wrangler.jsonc`:

| Binding | Resource |
|---|---|
| `kv_namespaces` | Key-value edge cache |
| `r2_buckets` | Object storage (zero egress) |
| `d1_databases` | Serverless SQL |
| `queues` | Async message queue |
| `durable_objects` | Stateful singleton per name |
| `services` | Call another Worker |
| `ai` | Workers AI inference |
| `vars` | Plaintext env vars |
| `secrets` | Encrypted env vars |

This skill does **not** provision those resources — create them with the corresponding `wrangler` CLI subcommands first.

## TL;DR

```bash
# One-time login (OAuth, browser opens)
npx wrangler login

# Deploy (reads wrangler.jsonc / wrangler.toml)
npx wrangler deploy

# → https://<worker-name>.<account-subdomain>.workers.dev
```

For SSH / headless machines: `npx wrangler login --device` instead of `wrangler login`.

## Source

The actual skill definition is [`SKILL.md`](./SKILL.md). Install by copying it into your Hermes skills directory:

```bash
mkdir -p ~/.hermes/skills/devops/cloudflare-workers-deploy
cp SKILL.md ~/.hermes/skills/devops/cloudflare-workers-deploy/
```
