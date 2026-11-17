# GitHub Actions deployment

The workflow at `.github/workflows/deploy.yml` runs on every push to `main` and can
also be started manually from **GitHub → Actions → Deploy RecFlare to Cloudflare**.
It installs dependencies, runs the repository checks, applies migrations, and deploys
all Workers with `just deploy`.

## 1. Prepare Cloudflare once

This repository is a multi-Worker monorepo, not a single Worker. Before the first
workflow run, prepare the Cloudflare resources described in [DEPLOYING.md](DEPLOYING.md):

- A Cloudflare account and a domain/zone you control.
- One D1 database named `recflare`.
- KV namespaces named `RECFLARE_MATCH_PRESENCE` and `RECFLARE_PLAYER_SETTINGS`.
- A Workers Secrets Store named `recflare`.
- R2 buckets named `recflare-cdn` and `recflare-img`.

Authenticate Wrangler locally, then create the resources:

```sh
wrangler login
wrangler d1 create recflare
wrangler kv namespace create RECFLARE_MATCH_PRESENCE
wrangler kv namespace create RECFLARE_PLAYER_SETTINGS
wrangler secrets-store store create recflare --scopes workers
wrangler r2 bucket create recflare-cdn
wrangler r2 bucket create recflare-img
```

Save the D1 database ID, both KV namespace IDs, and the Secrets Store ID from the
command output. The KV IDs must be stored together as compact JSON, for example:

```json
{"RECFLARE_MATCH_PRESENCE":"<namespace-id>","RECFLARE_PLAYER_SETTINGS":"<namespace-id>"}
```

Create the shared secret-store entries required by the workers. Use strong, unique
values for `JWT_SECRET`; set `META_APP_SECRET` to your Meta app secret (or a
placeholder if you do not use Meta login); and set `CLOUD_BUILD_KEY` to `off`
if you are not running the separate Studio cloud-build host:

```sh
wrangler secrets-store secret create <store-id> --name JWT_SECRET --scopes workers --remote
wrangler secrets-store secret create <store-id> --name META_APP_SECRET --scopes workers --remote
wrangler secrets-store secret create <store-id> --name CLOUD_BUILD_KEY --scopes workers --remote
```

Follow the remaining one-time setup in [DEPLOYING.md](DEPLOYING.md), including Cloudflare
Email Sending and any optional Turnstile/Discord integrations you want to enable.
Photon Realtime/Voice/Chat and a Tachyon voice server are separate game-service
dependencies; deploying Workers does not provision those services.

## 2. Add GitHub Actions secrets

In **GitHub → arttursedig26-create/server → Settings → Secrets and variables →
Actions → New repository secret**, add:

| Secret name | Value |
| --- | --- |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token with the permissions required by Wrangler to deploy Workers and manage the project's D1, KV, R2, and related resources. |
| `CLOUDFLARE_ACCOUNT_ID` | Your Cloudflare account ID. |
| `RECFLARE_DOMAIN` | Your domain only, e.g. `example.com` (no protocol or path). |
| `RECFLARE_D1` | The D1 database ID for `recflare`. |
| `RECFLARE_KV` | The compact JSON object containing both KV namespace IDs. |
| `RECFLARE_SECRETS_STORE` | The ID of the Workers Secrets Store. |

Never commit `.env`, API tokens, signing keys, or other credentials to Git.

## 3. Run deployment

After the resources and six GitHub secrets are ready, push a commit to `main`, or
open **Actions → Deploy RecFlare to Cloudflare → Run workflow**. The workflow stops
with a clear missing-secret message if any of the six GitHub secrets is absent.

Database migrations are applied before deployment, as required by the repository's
deployment instructions. Review database changes before running this workflow against
a production database.
