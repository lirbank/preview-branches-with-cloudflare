# Local dev

1. `cp .env.example .env`
1. Create a Neon project in "AWS US East 1 (N. Virginia)"
1. Create a Neon branch for development "dev"
1. Copy the pooled connection string to `DATABASE_URL` in `.env`
1. Copy the unpooled connection string to `CLOUDFLARE_HYPERDRIVE_LOCAL_CONNECTION_STRING_HYPERDRIVE` and `DATABASE_URL_UNPOOLED` in `.env`
1. Now you can run the app locally with `wrangler dev`

# CF

1. Go to **Workers & Pages** in CF Dashboard, then _Create application_ → _Continue with GitHub_ → and follow the steps to link your git repo. On the _Set up your application_ step, set _Deploy command_ and _Non-production branch deploy command_ to `scripts/deploy.ts`, and click _Deploy_. (will fail)
1. Create Neon API key for your Neon project.
1. In CF Dashboard, create a secret under _Build_ (not under _Variables and Secrets_) called `NEON_PROJECT_ID` and add the Neon API key.
1. In CF Dashboard, create a variable under _Build_ (not under _Variables and Secrets_) called `NEON_API_KEY` and set it to you Neon project ID.
1. Rebuild the CF Worker, it should now work
