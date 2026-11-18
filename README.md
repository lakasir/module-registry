# Lakasir Module Registry

The community index of installable Lakasir modules.

Lakasir reads `index.json` from this repository and shows its contents under
**Setting → Modules → Catalog**. Installing a module from there downloads the
release archive you point at, runs `composer install`, and migrates it on every
tenant — no server of ours is involved beyond this file.

## Contents

```
index.json                       the catalogue (Lakasir reads this file)
README.md                        this document
schemas/index.schema.json        JSON Schema for index.json
.github/workflows/validate.yml   CI that validates every pull request
```

## Adding a module

1. Publish a **release** of your module repository with a version tag
   (`v1.2.0`) and a source archive.
2. Fork this repository and add one entry to the `modules` array in
   `index.json`, keeping the list sorted by `name`.
3. Open a pull request. CI validates the file and checks that your archive is
   reachable.

A maintainer reviews it. Once merged, the module appears in every Lakasir
installation's Catalog the next time the Catalogue is opened; the file is
fetched on demand and there is no cache to wait out.

## The manifest of a module

A module is a zip archive that unpacks to *one* top-level directory containing:

```
composer.json                      required — "name": "vendor/package"
src/<Folder>ServiceProvider.php   required — e.g. src/Warehouse/WarehouseServiceProvider.php
src/Filament/Resources/...        optional — resources appear in the panel
database/migrations/...           optional — run in each tenant database
routes/web.php                    optional — mounted under modules/<module-slug>
lang/, resources/views/, config/  optional
```

`<Folder>` is the studly-cased package name: `lakasir/warehouse` → `Warehouse`.
Lakasir refuses an archive whose provider does not match that rule.

GitHub's "Download ZIP" of a tag already has the single top-level directory the
installer expects.

## Entry format

```json
{
  "name": "lakasir/warehouse",
  "folder": "Warehouse",
  "title": "Warehouse",
  "description": "Raw materials, production recipes and batches with HPP costing.",
  "author": "lakasir",
  "version": "1.2.0",
  "repository": "https://github.com/lakasir/warehouse-module",
  "download": "https://github.com/lakasir/warehouse-module/archive/refs/tags/v1.2.0.zip",
  "checksum": "sha256:9f2c…",
  "min_app_version": "1.1.0",
  "license": "GPL-3.0-only",
  "screenshots": ["https://raw.githubusercontent.com/lakasir/warehouse-module/main/docs/recipe.png"]
}
```

| Field | Required | Notes |
|---|---|---|
| `name` | yes | Composer package name, `vendor/package`. Must match the archive's `composer.json`. |
| `folder` | yes | Directory under `modules/`. Must equal the studly-cased package name. |
| `title` | yes | Display name in the Catalog. |
| `description` | yes | One or two sentences. Shown on the card. |
| `author` | yes | Your GitHub handle or organisation. |
| `version` | yes | Semver of the release, matching the tag. |
| `repository` | yes | Source repository, for reference and support. |
| `download` | yes | Direct URL to the release archive (`.zip`). Must be reachable without authentication. |
| `checksum` | recommended | `sha256:<hex>`. Lakasir refuses an archive whose hash differs. |
| `min_app_version` | optional | Oldest Lakasir that may install it, e.g. `"1.1.0"`. |
| `license` | optional | SPDX identifier. |
| `screenshots` | optional | Up to three absolute URLs. |

### Producing the checksum

```bash
curl -L -o module.zip https://github.com/you/module/archive/refs/tags/v1.2.0.zip
sha256sum module.zip          # macOS: shasum -a 256 module.zip
```

GitHub's generated archives are not guaranteed byte-stable over time, so the
checksum must be of the exact file the `download` URL served on release day. If
it later drifts, reopen this file with the new value.

## Local development

Modules under active development should **not** come through here. Check yours
out as a git submodule instead — Lakasir's installer refuses to overwrite a git
checkout, and a submodule keeps your working tree intact:

```bash
git submodule add git@github.com:you/your-module.git modules/YourModule
(cd modules/YourModule && composer install)
```

Publish to the registry once the module is ready for people who only want to
install a release.

## Rules

- One entry per package name; duplicates fail CI.
- `download` must serve a zip, not an HTML page or a login wall.
- Modules must not require a paid licence key to install.
- A module abandoned for twelve months may be moved to `deprecated.json`; it
  then stops appearing in the Catalog.
- The maintainers may decline a module that runs destructive migrations without
  a rollback path.

## Security

Every module in this registry executes PHP on the installer's server. The
`checksum` field is the only integrity guarantee between release and install, so
please publish one. Lakasir validates the archive (path traversal, manifest,
provider class) *before* extraction, and refuses to touch git checkouts — but it
cannot audit your code. That review is ours, which is why every entry arrives by
pull request.

## Removing a module

Send a pull request deleting the entry, or file an issue with the package name.
Removal from the index does not uninstall the module from anyone's installation;
it only hides it from future Catalogs.
