# cf-deploy

Shared GitHub Actions workflows for deploying to Cloudflare. Each repo keeps a
small `deploy.yml` that calls one of these, so a change on Cloudflare's side
(like the retirement of `cloudflare/pages-action`) is fixed here once instead
of in every repo.

| Workflow | Deploys | Pull requests |
|---|---|---|
| `pages.yml` | a static site to a Pages project | build only |
| `worker.yml` | a Worker, with optional pre-deploy steps (D1 migrations) | `wrangler deploy --dry-run` |

Pull requests never get the Cloudflare secrets (Dependabot's included), so
they only build. Pushes and manual runs deploy. Without a
`CLOUDFLARE_API_TOKEN` secret, the workflow builds and warns instead of
failing, so a fork or template still has a working check.

## Pages

```yaml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    uses: ry-ops/cf-deploy/.github/workflows/pages.yml@<sha> # v1
    with:
      project: my-site
      account-id: <account id>
      directory: dist
      # install-command: npm ci --legacy-peer-deps
      # build-env: |
      #   VITE_API_URL=https://api.example.com
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

`pages.yml` runs wrangler through `npx`, never installing it into your
project, and creates the Pages project on the first deploy if it doesn't
exist. The deployment URL is a job output (`url`) and goes in the run
summary.

## Worker

```yaml
jobs:
  deploy:
    uses: ry-ops/cf-deploy/.github/workflows/worker.yml@<sha> # v1
    with:
      account-id: <account id>
      pre-deploy: npx wrangler d1 migrations apply my-db --remote
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

`worker.yml` uses the wrangler version your project installs, so the deploy
matches what you test locally.

## Tokens

Give each repo its own API token, limited to what it deploys: **Account →
Cloudflare Pages → Edit** for a site, **Workers Scripts → Edit** (plus D1 if
it migrates) for a Worker. One leaked token then exposes one project.

## Pinning

Pin callers to a commit SHA. Dependabot's `github-actions` updates bump
reusable-workflow references like any other action.
