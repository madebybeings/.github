# madebybeings/.github

Shared automation for Made by Beings repositories.

This repository contains reusable GitHub Actions workflows and composite actions used across client sites, internal products and WordPress releases. Keep cross-repository CI behaviour here where possible, and keep only project-specific orchestration in each caller repository.

## `site-ci.yml` reusable workflow

`./.github/workflows/site-ci.yml` is the standard quality pipeline for Beings Next.js websites.

It centralises the repeated CI setup used across the website fleet:

1. Checkout.
2. pnpm setup.
3. Node setup from the caller repo's `.nvmrc`.
4. Frozen dependency install.
5. Optional Sanity environment wiring.
6. Optional build-environment preparation.
7. Typecheck.
8. Tailwind audit.
9. Optional project-specific blocking audits.
10. Optional non-blocking audits.
11. Production build.

It also applies per-repository and per-ref concurrency cancellation, so a newer commit cancels an obsolete CI run on the same ref.

### Standard caller

A normal site should keep its triggers in its own `.github/workflows/ci.yml` and delegate the quality job to the shared workflow:

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main
      - dev

jobs:
  quality:
    uses: madebybeings/.github/.github/workflows/site-ci.yml@main
```

The defaults assume:

```text
pnpm 11.1.3
Node version from .nvmrc
node scripts/write-ci-build-stub.mjs
pnpm typecheck
pnpm audit:tailwind
pnpm build
```

The Tailwind audit is soft by default so active-development sites still report design-system violations without blocking the build.

### Standard site with dev deployment

Deployment stays in the caller repository because the slug, port, command and checkout path are project-specific:

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main
      - dev

jobs:
  quality:
    uses: madebybeings/.github/.github/workflows/site-ci.yml@main

  deploy-dev:
    name: Deploy dev site
    needs: quality
    if: github.ref == 'refs/heads/dev' && github.event_name == 'push'
    uses: madebybeings/.github/.github/workflows/deploy-dev-site.yml@main
    with:
      slug: example-site
      port: 3005
      command: pnpm exec next dev -p 3005
      local_path: /home/coder/clients/example/site
      framework: next
    secrets:
      DEVCTL_SHARED_SECRET: ${{ secrets.DEVCTL_SHARED_SECRET }}
```

### Strict audits and project-specific checks

Projects can tighten the shared workflow without copying it.

For example, a site that requires Tailwind violations to fail CI and has additional brand audits can use:

```yaml
jobs:
  quality:
    uses: madebybeings/.github/.github/workflows/site-ci.yml@main
    with:
      prepare_command: ''
      tailwind_audit_mode: strict
      extra_audit_commands: |
        pnpm audit:em-dash:source
        pnpm audit:buttons
```

Use `extra_audit_commands` for checks that must block the build.

Use `soft_audit_commands` for useful diagnostics that should be reported but should not make CI fail:

```yaml
jobs:
  quality:
    uses: madebybeings/.github/.github/workflows/site-ci.yml@main
    with:
      soft_audit_commands: |
        pnpm audit:security
```

### Pre-install checks

If a project needs to validate its runtime before dependency installation, use `pre_install_command`:

```yaml
jobs:
  quality:
    uses: madebybeings/.github/.github/workflows/site-ci.yml@main
    with:
      pre_install_command: pnpm preflight:node
```

This is useful for projects that enforce an exact Node runtime or other repository-level preflight rules.

### Sanity builds

Most sites can rely on `scripts/write-ci-build-stub.mjs` and do not need real Sanity access during CI.

If a production build needs live Sanity reads, pass the public project metadata as inputs and the read token as a secret:

```yaml
jobs:
  quality:
    uses: madebybeings/.github/.github/workflows/site-ci.yml@main
    with:
      sanity_project_id: abc123
      sanity_dataset: production
      sanity_api_version: '2026-05-11'
    secrets:
      SANITY_API_READ_TOKEN: ${{ secrets.SANITY_API_READ_TOKEN }}
```

The shared workflow exposes these to the caller job as:

```text
NEXT_PUBLIC_SANITY_PROJECT_ID
NEXT_PUBLIC_SANITY_DATASET
NEXT_PUBLIC_SANITY_API_VERSION
SANITY_API_READ_TOKEN
```

Do not pass write tokens into normal CI unless a workflow genuinely needs to mutate content.

### Keeping specialised jobs local

The shared workflow is intentionally limited to normal site quality checks.

Keep jobs local when they are project-specific or materially different, for example:

```text
Playwright smoke and accessibility suites
GitLab mirrors
release workflows
production promotion
special deployment flows
large integration-test environments
```

A local specialised job can depend on the shared quality job normally:

```yaml
jobs:
  quality:
    uses: madebybeings/.github/.github/workflows/site-ci.yml@main
    with:
      tailwind_audit_mode: strict

  playwright:
    needs: quality
    runs-on: ubuntu-latest
    steps:
      # project-specific browser test setup
```

Voyager Tavern follows this pattern: its common quality stage is shared, while its Playwright smoke and accessibility job remains local.

### Inputs

| Input | Default | Purpose |
| --- | --- | --- |
| `pnpm_version` | `11.1.3` | pnpm version used by CI |
| `pre_install_command` | empty | Optional runtime/preflight check before install |
| `prepare_command` | `node scripts/write-ci-build-stub.mjs` | Prepare `.env.local` or other build state |
| `typecheck_command` | `pnpm typecheck` | Typecheck command |
| `tailwind_audit_command` | `pnpm audit:tailwind` | Tailwind/design token audit command |
| `tailwind_audit_mode` | `soft` | `soft`, `strict`, or `off` |
| `extra_audit_commands` | empty | Blocking project-specific checks |
| `soft_audit_commands` | empty | Non-blocking project-specific checks |
| `build_command` | `pnpm build` | Production build command |
| `sanity_project_id` | empty | Optional Sanity project ID |
| `sanity_dataset` | empty | Optional Sanity dataset |
| `sanity_api_version` | empty | Optional Sanity API version |
| `force_javascript_actions_to_node24` | `false` | Force JavaScript actions onto Node 24 when required |

### Secrets

| Secret | Required | Purpose |
| --- | --- | --- |
| `SANITY_API_READ_TOKEN` | No | Read-only Sanity access for builds that need live content |

### Changing the shared workflow

Treat changes to `site-ci.yml` as fleet-level infrastructure changes.

Before changing defaults, consider all current callers. A change to the shared workflow can affect Luna Park, Smile On, Lumitas, ALDI, the Beings website, Voyager Tavern, the boilerplate and future sites.

Prefer:

1. Backwards-compatible inputs for unusual project needs.
2. Safe defaults that fit normal active-development sites.
3. Project-specific checks in caller inputs rather than duplicated workflow files.
4. Local specialised jobs when a workflow is genuinely unique.

Avoid adding a check globally just because one site needs it.

## `deploy-dev-site.yml` reusable workflow

`./.github/workflows/deploy-dev-site.yml` deploys a Beings dev site by calling the `devctl` control plane on the Beings Development Server.

The caller supplies the site-specific slug, port, command and server checkout path. The repository URL is derived from the caller automatically.

This workflow should normally run only after the shared `quality` job succeeds on a push to `dev`.

## `wp-release` composite action

`./.github/actions/wp-release` builds a clean WordPress plugin/theme ZIP from a tagged commit and attaches it to the GitHub Release. It is the single artifact source of truth for the Beings WordPress plugin fleet, both the internal client fleet and the external Minimly licensing channel.

### How a repo uses it

1. Add a `.distignore` listing dev-only paths to exclude from the ZIP.
2. Add `.github/workflows/release.yml`:

   ```yaml
   name: Release
   on:
     push:
       tags: ["v*.*.*"]
   permissions:
     contents: write
   jobs:
     release:
       runs-on: ubuntu-latest
       steps:
         - uses: madebybeings/.github/.github/actions/wp-release@main
           with:
             slug: my-plugin-slug
   ```

3. To ship, bump the `Version:` header in the main plugin/theme file, commit, then `git tag vX.Y.Z && git push --tags`.

### What the ZIP contains

`rsync` of the repo minus hard excludes (`.git`, `.github`, `node_modules`, `*.zip`) and everything in the repo's `.distignore`.

Build steps run first: `npm ci && npm run build` when a `package.json` build script exists, and `composer install --no-dev` when a `composer.json` exists. This keeps runtime artifacts such as `build/` and `vendor/` while allowing source-only tooling to be excluded through `.distignore`.

### Versioning

The `Version:` header in the main PHP file, or `style.css` for themes, is canonical. The tag should match it, for example `v1.2.3` and `Version: 1.2.3`.

The Plugin Update Checker library on client sites compares the GitHub release tag against the installed header version.
