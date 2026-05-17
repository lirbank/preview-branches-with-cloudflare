# Preview branches with Cloudflare

This example gives every Cloudflare Workers preview deployment its own isolated [Neon](https://neon.tech/?ref=github) database branch, fronted by a dedicated [Cloudflare Hyperdrive](https://developers.cloudflare.com/hyperdrive/) config — created and wired up automatically on every push.

There are no GitHub Actions workflows. A single deploy script (`scripts/deploy.ts`) is run by [Cloudflare Workers Builds](https://developers.cloudflare.com/workers/ci-cd/builds/) and drives the whole flow: it creates (or reuses) the Neon branch, provisions Hyperdrive, injects the connection strings, and deploys the Worker — for both production and previews.

## How it works

Deploys are triggered by the [Cloudflare Workers and Pages GitHub App](https://github.com/apps/cloudflare-workers-and-pages), not by workflows in this repo. On every build, Workers Builds runs `scripts/deploy.ts`, which branches on the Git branch being built:

- Push to the default branch (`main`) → production deploy, paired with Neon's default branch.
- Push to any other branch → preview deploy, backed by a Neon branch dedicated to that Git branch and reused across commits.

The connection string reaches the Worker as `env.HYPERDRIVE` (Hyperdrive binding) and as plain `env.DATABASE_URL` (pooled) and `env.DATABASE_URL_UNPOOLED` (unpooled) values.

### Preview flow

1. Push to a non-default Git branch.
2. The Cloudflare GitHub App triggers a non-production build, which runs `bun run scripts/deploy.ts`.
3. The script deletes any Hyperdrive config whose Neon branch no longer exists.
4. It looks up the Neon branch named `preview-<branch-slug>`, reusing it if present or creating it with a 7-day TTL.
5. It fetches the branch's pooled and unpooled connection URIs.
6. It upserts a Hyperdrive config named `preview-branches-with-cloudflare--preview--<branch-slug>` pointing at the unpooled URI.
7. It runs `wrangler versions upload --preview-alias <branch-slug>` with the Hyperdrive binding and the connection strings bundled in.
8. When the Git branch is deleted, the Neon branch expires via its TTL, and the next deploy removes the orphaned Hyperdrive config.

### Production flow

1. Push to the default Git branch (`main`).
2. The Cloudflare GitHub App triggers a production build, which runs `bun run scripts/deploy.ts`.
3. The script deletes any Hyperdrive config whose Neon branch no longer exists.
4. It finds Neon's default branch and fetches its pooled and unpooled connection URIs.
5. It upserts a Hyperdrive config named `preview-branches-with-cloudflare--production` pointing at the unpooled URI.
6. It runs `wrangler deploy` with the Hyperdrive binding and the connection strings bundled in.

Production credentials are fetched from Neon on every deploy.

## Tech stack

- [Neon](https://neon.tech/?ref=github) — managed Postgres with database branching
- [Cloudflare Workers](https://developers.cloudflare.com/workers/) — deployment platform
- [Cloudflare Workers Builds](https://developers.cloudflare.com/workers/ci-cd/builds/) — CI/CD, triggered by the Cloudflare GitHub App
- [Cloudflare Hyperdrive](https://developers.cloudflare.com/hyperdrive/) — connection pooling in front of Postgres
- [node-postgres (`pg`)](https://node-postgres.com/) — Postgres client
- [Bun](https://bun.sh) — runtime and package manager

This example ships a minimal Worker that runs one sample query. It has no schema, migrations, or seed data — it demonstrates the preview-branching and connection-injection mechanism, not application code.

## Prerequisites

- A [Neon project](https://neon.tech/?ref=github) (the free tier is enough).
- A Neon API key with permission to create branches and read connection strings — see [Manage API keys](https://neon.tech/docs/manage/api-keys).
- Your Neon project ID.
- A Cloudflare account.
- The [Cloudflare Workers and Pages GitHub App](https://github.com/apps/cloudflare-workers-and-pages) installed on your fork of this repository.

You do not need to create a Cloudflare API token by hand. Workers Builds injects `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` automatically, with the scopes needed to manage Hyperdrive.

## Setup

1. Clone the repository and install dependencies (you need [Bun](https://bun.sh)):

   ```bash
   git clone https://github.com/neondatabase/preview-branches-with-cloudflare.git
   cd preview-branches-with-cloudflare
   bun install
   ```

2. Set `placement.region` in [`wrangler.jsonc`](wrangler.jsonc) to the region closest to your Neon database (e.g. `aws:us-east-1`).

3. Create a Worker on Cloudflare and connect this repository to it via the Cloudflare Workers and Pages GitHub App, so deployments run through Workers Builds.

   > **VERIFY:** exact dashboard path (Workers & Pages → Create → connect to Git) — confirm when walking the steps.

4. In the Worker's Workers Builds settings, add these build environment variables:

   | Variable             | Type   | Required | Value                                                    |
   | -------------------- | ------ | -------- | -------------------------------------------------------- |
   | `NEON_API_KEY`       | Secret | Yes      | Neon API key with branch create + connection-string read |
   | `NEON_PROJECT_ID`    | Text   | Yes      | Your Neon project ID                                     |
   | `GIT_DEFAULT_BRANCH` | Text   | No       | Git default branch name (defaults to `main`)             |

   > **VERIFY:** location and the Secret/Text distinction for build variables — confirm in the dashboard.

5. Set both deploy commands to the deploy script:
   - Production deploy command: `bun run scripts/deploy.ts`
   - Non-production deploy command: `bun run scripts/deploy.ts`

   The script decides production vs. preview internally from `WORKERS_CI_BRANCH`.

   > **VERIFY:** exact field names for the production / non-production deploy commands — confirm in the dashboard.

6. Push to `main` for the first production deploy. Then open a branch to get a preview deployment with its own Neon branch.

   > **VERIFY:** the preview URL format produced by `wrangler versions upload --preview-alias`.

## Local development

```bash
cp .env.example .env
```

Fill in the connection string(s) in `.env`, then start the dev server:

```bash
bun run dev
```

After changing bindings in [`wrangler.jsonc`](wrangler.jsonc), regenerate types:

```bash
bun run cf-typegen
```

> **VERIFY:** that `bun run dev` picks up the local Hyperdrive connection string from `.env`.

Do not run `wrangler deploy` locally. A local deploy would inherit the latest Worker version's secret (typically the most recent preview's database URL) and clobber production. All deploys go through Workers Builds.

## Notes and limitations

- One Neon branch per Git branch, reused across commits. Test data created during review persists into the next preview deploy. A force-push that rewrites schema-changing commits does not rewind the database.
- Cleanup is TTL-driven. Abandoned Neon branches expire after 7 days; the TTL is refreshed on every deploy. Orphaned Hyperdrive configs are removed at the start of the next deploy — long-idle projects keep orphans until something deploys again.
- Non-default branches always parent on Neon's default branch, so a stacked branch does not see an un-merged parent's schema.
- On the first deploy of a new Hyperdrive config, the binding may not appear in the Cloudflare dashboard's Bindings tab for a deploy or two. The runtime is unaffected.

## Resources

- [Neon documentation](https://neon.tech/docs)
- [Neon branching](https://neon.tech/docs/introduction/branching)
- [Cloudflare Workers Builds](https://developers.cloudflare.com/workers/ci-cd/builds/)
- [Cloudflare Hyperdrive](https://developers.cloudflare.com/hyperdrive/)
