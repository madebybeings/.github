# madebybeings/.github

Org-level shared automation for Made by Beings repos.

## `wp-release` composite action

`./.github/actions/wp-release` builds a clean WordPress plugin/theme ZIP from a
tagged commit and attaches it to the GitHub Release. It is the single artifact
source of truth for the Beings WordPress plugin fleet — both the internal client
fleet (which reads GitHub Releases directly via a PAT) and the external Minimly
licensing channel (minimly.io redirects to the same release asset).

### How a repo uses it

1. Add a `.distignore` listing dev-only paths to exclude from the ZIP (WP-standard).
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
             slug: my-plugin-slug   # = install folder name
   ```

3. To ship: bump the `Version:` header in the main plugin/theme file, commit,
   then `git tag vX.Y.Z && git push --tags`. The action builds the ZIP and
   publishes the release.

### What the ZIP contains

`rsync` of the repo minus hard excludes (`.git`, `.github`, `node_modules`,
`*.zip`) and everything in the repo's `.distignore`. Build steps run first:
`npm ci && npm run build` when a `package.json` build script exists, and
`composer install --no-dev` when a `composer.json` exists. So `build/` and
`vendor/` are present in the ZIP; `src/`, lockfiles, and tooling configs are not
(when listed in `.distignore`).

### Versioning

The `Version:` header in the main PHP file (or `style.css` for themes) is the
canonical version. The tag should match (`v1.2.3` ↔ `Version: 1.2.3`). The
Plugin Update Checker library on client sites compares the GitHub release tag
against the installed header version.
