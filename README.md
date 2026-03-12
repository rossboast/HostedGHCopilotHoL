# HostedGHCopilotHoL

Hosted deployment of the **GitHub Copilot Hands-on Lab** workshop, built on top of [Microsoft MOAW](https://github.com/microsoft/moaw) (Microsoft Open-source Apps for Workshops) and deployed to GitHub Pages.

**Live site:** `https://<owner>.github.io/HostedGHCopilotHoL/`

## Overview

This repository self-hosts a single workshop from [Philess/GHCopilotHoL](https://github.com/Philess/GHCopilotHoL) using the MOAW platform as the rendering engine. A GitHub Actions workflow automatically syncs both the MOAW engine and workshop content from their upstream repositories, applies the necessary patches to make everything work as a standalone GitHub Pages deployment, and then builds and deploys the site.

The key challenge is that MOAW is designed to run at `moaw.dev` (or under the `/moaw/` path on `github.io`), and the workshop content is designed to be consumed externally via `moaw.dev` links. This repository patches both the engine and content so that the workshop is served directly as a local, self-contained site.

## How It Works

### Architecture

```
┌─────────────────────────┐    ┌──────────────────────────┐
│  microsoft/moaw (engine)│    │ Philess/GHCopilotHoL     │
│  - packages/website     │    │ (workshop content)        │
│  - packages/database    │    │  - docs/workshop.md       │
│  - packages/cli         │    │  - docs/assets/           │
└──────────┬──────────────┘    └──────────┬───────────────┘
           │                              │
           │     GitHub Actions Workflow  │
           └──────────┐  ┌───────────────┘
                      ▼  ▼
              ┌───────────────────┐
              │  Sync Job         │
              │  - Copies engine  │
              │  - Copies content │
              │  - Applies patches│
              │  - Commits to repo│
              └────────┬──────────┘
                       ▼
              ┌───────────────────┐
              │  Deploy Job       │
              │  - npm ci         │
              │  - create:db      │
              │  - build:website  │
              │  - GitHub Pages   │
              └───────────────────┘
                       ▼
              ┌───────────────────┐
              │  GitHub Pages     │
              │  /HostedGHCopilotHoL/ │
              └───────────────────┘
```

### Content Pipeline

1. **Sync job** checks out the MOAW engine (`microsoft/moaw`) and workshop content (`Philess/GHCopilotHoL`) into temporary directories
2. The engine's `packages/`, `package.json`, and `package-lock.json` are copied into this repo, replacing the previous versions
3. Workshop content from `docs/` is copied into `workshops/github-copilot/`
4. A series of `sed` patches are applied (see [Patches](#patches-applied-during-sync) below)
5. Changes are committed and pushed

6. **Deploy job** picks up the committed content and:
   - Runs `npm ci` to install dependencies (npm workspaces)
   - Runs `npm run create:db` — the database package scans `workshops/` and generates `workshops.json`, which is the catalog the Angular app uses to list and link workshops
   - Runs `npm run build:website` — builds the Angular SPA with the correct `--base-href`
   - Uploads the built site via `upload-pages-artifact` and deploys to GitHub Pages

### URL Resolution

The MOAW website is an Angular SPA. When a user clicks a workshop card:

1. The app reads entries from `workshops.json`
2. Each entry has a `url` field — for local workshops this is a relative path like `github-copilot/`
3. The app checks if the URL starts with `http` — if so, it's treated as an external link; otherwise it's resolved relative to the site's `<base href>`
4. The `<base href>` is set both statically in `index.html` and dynamically via runtime JavaScript that checks `window.location.hostname`
5. The Angular router uses `document.baseURI` to derive the base path for all routes

This is why getting the base href right is critical — if it points to `/moaw/` instead of `/HostedGHCopilotHoL/`, all navigation and workshop links break.

## Patches Applied During Sync

The sync job applies several `sed` patches to transform the upstream sources into a working standalone deployment. Each patch addresses a specific incompatibility:

### 1. Set `published: true` in workshop frontmatter

```bash
sed -i 's/^published: false/published: true/' "$workshop_file"
```

The upstream workshop has `published: false` because it's meant to be consumed via external MOAW links. The database generator only includes workshops with `published: true` in the catalog, so this patch is required for the workshop to appear on the site.

### 2. Remove `moaw.dev` link from workshop content

```bash
sed -i '/moaw\.dev\/workshop\/gh:Philess\/GHCopilotHoL/d' "$workshop_file"
```

The upstream workshop markdown contains a link to itself on `moaw.dev`. Since we're hosting locally, this link would be confusing and point users away from the local deployment.

### 3. Fix Spanish translation links

```bash
sed -i 's|https://moaw\.dev/workshop/gh:Philess/GHCopilotHoL/main/docs/|../|g' "$es_file"
```

The Spanish translation references the workshop via a full `moaw.dev` URL. This replaces it with a relative path so it works locally.

### 4. Remove duplicate external entry

```bash
sed -i '/^- title: GitHub Copilot, your new AI pair programmer - The Ultimate workshop/,/^$/d' packages/database/external.yml
```

The MOAW engine's `external.yml` contains a curated list of externally hosted workshops, including this one pointing to `moaw.dev`. Without this patch, the catalog would show a duplicate entry — one local (from the `workshops/` scan) and one external (from `external.yml`) pointing to `moaw.dev`.

### 5. Fix base href for GitHub Pages (runtime override)

```bash
sed -i "s|'/moaw/'|'/HostedGHCopilotHoL/'|" packages/website/src/public/index.html
```

The MOAW `index.html` contains runtime JavaScript that detects `.github.io` domains and overrides the `<base href>` to `/moaw/`. This patch changes it to `/HostedGHCopilotHoL/` so all routes and assets resolve correctly under this repository's GitHub Pages path.

### 6. Add `--base-href` to Angular build

```bash
sed -i 's|"build": "ng build &&|"build": "ng build --base-href /HostedGHCopilotHoL/ \&\&|' packages/website/package.json
```

Angular CLI requires the `--base-href` flag at build time to embed the correct base path into the built application. The upstream build script doesn't include this flag (it relies on the runtime JS override). This patch ensures the Angular build produces correct asset paths for this deployment.

## Repository Structure

```
.github/workflows/
  sync-and-deploy.yml    # Main CI/CD workflow (sync + build + deploy)

packages/                # Synced from microsoft/moaw (replaced on each sync)
  website/               # Angular 17 SPA — the MOAW web application
  database/              # Workshop catalog generator (workshops.json)
  cli/                   # MOAW CLI tool

workshops/
  github-copilot/        # Synced from Philess/GHCopilotHoL
    workshop.md          # Main workshop content (English)
    assets/              # Workshop images and resources
    translations/        # Translated workshop content
      workshop.es.md     # Spanish translation
```

> **Note:** The `packages/`, `package.json`, and `package-lock.json` files are overwritten on each sync from upstream MOAW. Do not make manual edits to these files — any changes will be lost. All customizations must be applied as `sed` patches in the workflow.

## Prerequisites

To deploy your own instance:

1. **GitHub Pages** must be enabled in the repository settings, with the source set to **GitHub Actions**
2. The repository must be **public** (or have GitHub Pages available on your plan)
3. No secrets or environment variables are required — the `SEARCH_URL` used by the upstream MOAW deployment is not needed (search is gracefully disabled when empty)

## Workflow Triggers

The workflow runs on:
- **Push to `main`** — any commit triggers a full sync and deploy
- **Manual dispatch** — can be triggered manually from the Actions tab to pick up upstream changes

## Adapting for a Different Repository Name

If you fork or clone this to a repository with a different name, update the two `sed` patches that reference `HostedGHCopilotHoL`:

1. In the `index.html` base href patch, change `'/HostedGHCopilotHoL/'` to `'/<your-repo-name>/'`
2. In the `--base-href` patch, change `/HostedGHCopilotHoL/` to `/<your-repo-name>/`

## Adapting for Different Workshop Content

To host a different workshop:

1. Update the `UPSTREAM_REPOSITORY` env var to point to the content source repository
2. Update `CONTENT_SOURCE_PATH` if the content lives in a different directory
3. Update `TARGET_WORKSHOP_PATH` to match the desired URL path
4. Review and adjust the `sed` patches — the content-specific patches (published flag, moaw.dev link removal, translation fixes) will need to match the new content
5. Update or remove the `external.yml` patch if the new workshop isn't listed there
