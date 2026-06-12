# Cloudflare Pages Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the Zola site with Nix-provided `zola` and deploy it to Cloudflare Pages on every push to `main`, via a single GitHub Actions workflow.

**Architecture:** One workflow file (`.github/workflows/deploy.yml`) with a single `deploy` job: checkout → install Nix → `nix develop --command zola build` → `cloudflare/wrangler-action@v3` runs `wrangler pages deploy public --project-name=website`. Production only, triggered on push to `main` plus manual dispatch.

**Tech Stack:** GitHub Actions, Nix flakes (`DeterminateSystems/nix-installer-action`), Zola, Cloudflare Pages (`cloudflare/wrangler-action@v3`).

---

## Reference

- Spec: `docs/superpowers/specs/2026-06-13-cloudflare-pages-deploy-design.md`
- Existing flake devShell already provides `zola` (see `flake.nix`); the workflow reuses it so CI and local builds match.
- Cloudflare project name: `website` (already exists, direct-upload type).
- Required GitHub repo secrets (provisioned by the user, NOT created in this plan): `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`.

## File Structure

- **Create:** `.github/workflows/deploy.yml` — the entire deployment pipeline. Single responsibility: build and deploy on push to `main`.

No other files change.

---

### Task 1: Verify the build command works under Nix locally

This is the command the workflow will run. Confirm it succeeds before encoding it in CI, so a CI failure later points at the workflow config, not the build itself.

**Files:**
- None (verification only)

- [ ] **Step 1: Run the exact build command the workflow will use**

Run: `nix develop --command zola build`
Expected: command exits 0 and `public/` is (re)generated. If `direnv`/flake is already active, `zola build` alone also works, but run it through `nix develop` to match CI exactly.

- [ ] **Step 2: Confirm output exists**

Run: `test -f public/index.html && echo OK`
Expected: prints `OK`.

---

### Task 2: Create the deployment workflow

**Files:**
- Create: `.github/workflows/deploy.yml`

- [ ] **Step 1: Write the workflow file**

Create `.github/workflows/deploy.yml` with exactly this content:

```yaml
name: Deploy to Cloudflare Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: deploy-pages
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Nix
        uses: DeterminateSystems/nix-installer-action@main

      - name: Build site
        run: nix develop --command zola build

      - name: Deploy to Cloudflare Pages
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: pages deploy public --project-name=website
```

- [ ] **Step 2: Validate the YAML parses**

Run: `nix develop --command sh -c 'python3 -c "import yaml,sys; yaml.safe_load(open(\".github/workflows/deploy.yml\"))" && echo VALID'`
Expected: prints `VALID`. (If `python3` with PyYAML is unavailable in the shell, instead run `git hash-object .github/workflows/deploy.yml >/dev/null && echo "file written"` and rely on GitHub's own workflow parsing on push.)

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/deploy.yml
git commit -m "Add Cloudflare Pages deploy workflow"
```

---

### Task 3: Confirm prerequisites before first run (manual, no code)

This task is a checklist the user completes in the GitHub UI. The workflow will fail at the deploy step without it.

**Files:**
- None

- [ ] **Step 1: Confirm repo secrets exist**

In GitHub → Settings → Secrets and variables → Actions, confirm both exist:
- `CLOUDFLARE_API_TOKEN` — token scoped to *Account → Cloudflare Pages → Edit*.
- `CLOUDFLARE_ACCOUNT_ID`.

- [ ] **Step 2: Confirm the Pages project name matches**

Confirm the Cloudflare Pages project is named exactly `website`. If not, update the `--project-name=` value in `.github/workflows/deploy.yml` to match and commit the change.

---

### Task 4: End-to-end verification

**Files:**
- None

- [ ] **Step 1: Merge to main / push and watch the run**

After this branch merges to `main` (or via `workflow_dispatch` once the workflow is on `main`), open GitHub → Actions → "Deploy to Cloudflare Pages".
Expected: all four steps succeed; the deploy step logs a Cloudflare Pages deployment URL.

- [ ] **Step 2: Confirm the live site updated**

Run: `curl -sSI https://uspenskiy.tech | head -1`
Expected: `HTTP/2 200`. Spot-check the site renders the latest content.

---

## Self-Review

- **Spec coverage:** Trigger (push to main + dispatch) → Task 2 Step 1. Nix install + `nix develop` build → Task 2 + verified in Task 1. `wrangler-action` deploy to project `website` → Task 2. Secrets `CLOUDFLARE_API_TOKEN`/`CLOUDFLARE_ACCOUNT_ID` → Task 3. `contents: read` permission → Task 2. Out-of-scope items (previews, base_url override, caching, comments) correctly absent. End-to-end check → Task 4. All spec sections covered.
- **Placeholders:** None — full YAML and exact commands provided.
- **Consistency:** Project name `website`, secret names, and `public/` output path are identical across the spec, workflow, and tasks.
