# Cloudflare Pages Deployment via GitHub Actions

**Date:** 2026-06-13
**Status:** Approved
**Repo:** `wellWINeo/website` — Zola static site served at `https://uspenskiy.tech`

## Goal

Automatically build and deploy the site to Cloudflare Pages on every push to
`main`. The build uses Nix to provide `zola` (the same binary the local dev
shell uses), and deployment uses Cloudflare's officially maintained action.

## Decisions

- **Scope:** Production only. Deploy on push to `main`; no PR/branch preview
  deployments. This keeps the pipeline to a single linear job and avoids
  `base_url` overrides.
- **Cloudflare action:** `cloudflare/wrangler-action@v3` running
  `wrangler pages deploy`. Cloudflare's older `cloudflare/pages-action` is
  deprecated and archived, so it is not used.
- **Nix in CI:** `DeterminateSystems/nix-installer-action` installs Nix with
  flakes enabled. The build runs through the existing flake devShell via
  `nix develop --command zola build`, so CI and local dev use the exact same
  `zola` version pinned in `flake.lock`.
- **No Nix build cache:** `zola` is fetched from the public Nix binary cache
  (`cache.nixos.org`). For a site this small no extra caching layer is needed.
- **Pages project:** Already exists in Cloudflare, named `website`
  (direct-upload type). The project name is hardcoded in the workflow.

## Architecture

A single workflow file, `.github/workflows/deploy.yml`, with one job `deploy`
that runs on `ubuntu-latest`.

### Triggers

- `push` to `main`
- `workflow_dispatch` (manual re-run)

### Job permissions

- `contents: read` only. No PR comments or other write scopes are required for
  production-only deploys.

### Steps

1. **Checkout** — `actions/checkout@v4`.
2. **Install Nix** — `DeterminateSystems/nix-installer-action@main`.
3. **Build** — `nix develop --command zola build`. Produces `public/`.
   `base_url` stays `https://uspenskiy.tech` as configured in `zola.toml`.
4. **Deploy** — `cloudflare/wrangler-action@v3` with
   `command: pages deploy public --project-name=website`. Authenticates using
   the secrets below.

### Data flow

```
push to main
  → checkout repo
  → install nix (flakes)
  → nix develop --command zola build  →  public/
  → wrangler pages deploy public --project-name=website
  → live at https://uspenskiy.tech
```

## Configuration (set in GitHub repo settings)

Settings → Secrets and variables → Actions:

- `CLOUDFLARE_API_TOKEN` (secret) — scoped to *Account → Cloudflare Pages →
  Edit*.
- `CLOUDFLARE_ACCOUNT_ID` (secret).

These are referenced by the `wrangler-action` step. The user provisions them;
they are not stored in the repo.

## Error handling

- If `zola build` fails (broken template, invalid content), the job fails at the
  build step and nothing is deployed.
- If Cloudflare auth or the project name is wrong, the deploy step fails after a
  successful build; the previous live deployment is unaffected because Pages
  only promotes a new deployment on success.

## Out of scope (YAGNI)

- PR/branch preview deployments
- `base_url` overrides per environment
- Deployment status comments on commits/PRs
- Nix build caching (magic-nix-cache / FlakeHub Cache)
- `zola check` as a separate CI gate (can be added later if desired)

## Files changed

- **Add:** `.github/workflows/deploy.yml`

No other repository files change.
