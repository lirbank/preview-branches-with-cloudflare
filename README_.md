# Preview branches with Cloudflare

This is an example project that gives every Cloudflare Workers preview deployment its own isolated [Neon](https://neon.tech/?ref=github) database branch, fronted by a dedicated [Cloudflare Hyperdrive](https://developers.cloudflare.com/hyperdrive/) config — created and wired up automatically on every push.

There are **no GitHub Actions workflows**. A single deploy script (`scripts/deploy.ts`) is run by [Cloudflare Workers Builds](https://developers.cloudflare.com/workers/ci-cd/builds/) and drives the whole flow: it creates (or reuses) the Neon branch, provisions Hyperdrive, injects the connection strings, and deploys the Worker — for both production and previews.

## How it works

Deploys are triggered by the [Cloudflare Workers and Pages GitHub App](https://github.com/apps/cloudflare-workers-and-pages), not by workflows in this repo. On every build, Workers Builds runs `scripts/deploy.ts`, which branches on the Git branch being built:

- **Push to the default branch (`main`)** → production deploy, paired with Neon's default branch.
- **Push to any other branch** → preview deploy, backed by a Neon branch dedicated to that Git branch and reused across commits.

The connection string is delivered to the Worker two ways: as `env.HYPERDRIVE` (a Hyperdrive binding, recommended for pooled, low-latency access) and as plain `env.DATABASE_URL` / `env.DATABASE_URL_UNPOOLED` values for convenience.

### Preview flow

1. Push to a non-default Git branch.
2. The Cloudflare GitHub App triggers a non-production build, which runs `bun run scripts/deploy.ts`.
3. The script deletes any Hyperdrive config whose Neon branch no longer exists.
4. It looks up the Neon branch named `preview-<branch-slug>`, reusing it if present or creating it with a 7-day TTL.
5. It fetches the branch's pooled and unpooled connection URIs.
6. It upserts a Hyperdrive config named `<worker>--preview--<branch-slug>` pointing at the unpooled URI.
7. It writes a temporary wrangler config (injecting the Hyperdrive binding) and a temporary secrets file, then runs `wrangler versions upload --preview-alias <branch-slug>`.
8. When the Git branch is deleted upstream, the Neon branch expires on its own via the TTL, and the next deploy sweeps the orphaned Hyperdrive config.

### Production flow

1. Push to the default Git branch (`main`).
2. The Cloudflare GitHub App triggers a production build, which runs `bun run scripts/deploy.ts`.
3. The script deletes any Hyperdrive config whose Neon branch no longer exists.
4. It finds Neon's default branch and fetches its pooled and unpooled connection URIs.
5. It upserts a Hyperdrive config named `<worker>--production` pointing at the unpooled URI.
6. It writes a temporary wrangler config and secrets file, then runs `wrangler deploy`.

Production credentials are fetched fresh from Neon on every deploy, so rotating the Neon password is self-healing.

## Tech stack

- [Neon](https://neon.tech/?ref=github) — managed Postgres with database branching
- [Cloudflare Workers](https://developers.cloudflare.com/workers/) — deployment platform
- [Cloudflare Workers Builds](https://developers.cloudflare.com/workers/ci-cd/builds/) — CI/CD, triggered by the Cloudflare GitHub App
- [Cloudflare Hyperdrive](https://developers.cloudflare.com/hyperdrive/) — connection pooling and caching in front of Postgres
- [node-postgres (`pg`)](https://node-postgres.com/) — Postgres client
- [Bun](https://bun.sh) — runtime and package manager

## Prerequisites

- A [Neon project](https://neon.tech/?ref=github) (the free tier is enough).
- A Neon API key with permission to create branches and read connection strings — see [Manage API keys](https://neon.tech/docs/manage/api-keys).
- A Cloudflare account.
- The [Cloudflare Workers and Pages GitHub App](https://github.com/apps/cloudflare-workers-and-pages) installed on your fork of this repository.

You do **not** need to create a Cloudflare API token by hand. Workers Builds injects `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` automatically, with the scopes needed to manage Hyperdrive.

## Setup

1. Clone the repository:

   ```bash
   git clone https://github.com/neondatabase/preview-branches-with-cloudflare.git
   cd preview-branches-with-cloudflare
   ```

2. Install dependencies (you need [Bun](https://bun.sh)):

   ```bash
   bun install
   ```

3. Create the Worker on Cloudflare and connect this repository via the **Cloudflare Workers and Pages GitHub App** (Workers & Pages → Create → Workers → Connect to Git). This puts deployments under Workers Builds.

4. In the Worker's **Builds → Build configuration** settings, set the build environment variables:

   | Variable             | Type   | Required | Description                                              |
   | -------------------- | ------ | -------- | -------------------------------------------------------- |
   | `NEON_API_KEY`       | Secret | Yes      | Neon API key with branch create + connection-string read |
   | `NEON_PROJECT_ID`    | Text   | Yes      | Your Neon project ID                                     |
   | `GIT_DEFAULT_BRANCH` | Text   | No       | Git default branch name (defaults to `main`)             |

5. In the same settings, set **both** deploy commands to the deploy script:
   - Production deploy command: `bun run scripts/deploy.ts`
   - Non-production deploy command: `bun run scripts/deploy.ts`

   The script decides production vs. preview internally from `WORKERS_CI_BRANCH`.

6. Set `placement.region` in [`wrangler.jsonc`](wrangler.jsonc) to the region closest to your Neon database (e.g. `aws:us-east-1`). This keeps Worker-to-database latency low.

7. Push to `main` to trigger the first production deploy, then open a branch/PR to get a preview deployment with its own Neon branch.

## Local development

```bash
cp .env.example .env
```

Set the Hyperdrive local connection string in `.env` to a Neon branch's unpooled connection string, then start the dev server:

```bash
bun run dev
```

`wrangler dev` reads the local connection string from `.env` and serves the Worker against your Neon database. After changing bindings in [`wrangler.jsonc`](wrangler.jsonc), regenerate types:

```bash
bun run cf-typegen
```

> Do **not** run `wrangler deploy` locally. A local deploy would inherit the latest Worker version's secret (typically the most recent preview's database URL) and clobber production. All deploys must go through Workers Builds.

## Notes and limitations

- **One Neon branch per Git branch, reused across commits.** Test data created during review persists into the next preview deploy. A force-push that rewrites schema-changing commits does not automatically rewind the database.
- **Cleanup is TTL-driven.** Abandoned Neon branches expire after 7 days (the TTL is refreshed on every deploy). Orphaned Hyperdrive configs are removed at the start of the next deploy by cross-referencing live Neon branches — long-idle projects may keep orphans until something deploys again.
- **Stacked branches parent on Neon's default branch**, not on their Git parent's branch, so they do not see an un-merged parent's schema.
- **First deploy of a new Hyperdrive config** may not show the binding in the Cloudflare dashboard's Bindings tab for a deploy or two. The runtime is unaffected.
- **Schema and seed data are intentionally out of scope** for this example for now — it focuses on the preview-branching and connection-injection mechanism. Wiring migrations into the deploy step is tracked as a follow-up.

## Resources

- [Neon documentation](https://neon.tech/docs)
- [Neon branching](https://neon.tech/docs/introduction/branching)
- [Cloudflare Workers Builds](https://developers.cloudflare.com/workers/ci-cd/builds/)
- [Cloudflare Hyperdrive](https://developers.cloudflare.com/hyperdrive/)
