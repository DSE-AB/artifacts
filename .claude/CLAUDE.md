# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **binary artifact repository** for Pulsar Core and Pulsar SPA-UI, served as a raw-file CDN over GitHub at `https://github.com/DSE-AB/artifacts/raw/master/pulsar`. There is no source code to build here — the repo *publishes* `.pulsar`/`.jar` binaries (tracked via Git LFS, see `.gitattributes`) and the manifest `.properties` files that consumers resolve to download them.

The canonical consumer is `scripts/ant/pulsar-update.xml`, an Ant script that downstream Pulsar dev projects copy into their own tree to fetch a specific Core + SPA-UI combination.

## Layout

- `pulsar/core/<version>/` — Core release binaries: `pulsar.launcher-<launcherVer>.jar`, `pulsar-modules-<ver>.pulsar`, `pulsar-apps-<ver>.pulsar`, `pulsar-endorsed-<ver>.pulsar`, and a `Pulsar <ver> Documentation.pdf` for doc-bearing releases.
- `pulsar/spa-ui/<version>/` — Tagged SPA-UI releases: `spa-ui-<ver>.pulsar` + `spa-ui-endorsed-<ver>.pulsar`.
- `pulsar/spa-ui/latest/` — Rolling CI builds from the SPA-UI develop branch. Filenames carry build metadata: `spa-ui-<ver>.<build>.<YYMMDD-hhmm>-develop.pulsar`.
- `pulsar/*.properties` — **Manifests**, not configs. Each one names a release "channel" that a consumer can pin to.

## Manifest naming

| Pattern | Meaning |
| --- | --- |
| `pulsar-<core>.properties` | Core-only release (no SPA-UI). |
| `spa-ui-<spaui>-<core>.properties` | A tagged SPA-UI release pinned to a specific Core release. One file per supported pair. |
| `spa-ui-latest-<core>.properties` | Rolling SPA-UI develop build against a fixed Core release. Updated on every CI build worth publishing. |
| `spa-ui-latest-latest.properties` | Bleeding edge — rolling SPA-UI against rolling Core. |
| `pulsar-latest.properties` | Rolling Core only. |

A manifest's keys (`launcher`, `modules`, `apps`, `endorsed`, `docs`, `spaui`, `spaui-endorsed`) are paths *relative to `pulsar/`*. The Ant updater concatenates `${repositoryURL}` + path to GET each file.

## Conventions to follow when adding/updating manifests

- **`endorsed` is sticky.** Across SPA-UI manifests the endorsed package is almost always pinned to `core/4.14.0/pulsar-endorsed-4.14.0.pulsar` even when Core itself advances (4.16.x, 4.17.0, …). Don't auto-bump endorsed to match Core unless you've confirmed an actual endorsed release exists *and* should be picked up. A diff that "upgrades" endorsed to the current Core version is usually a regression from the RC phase, not an intentional change.
- **RC suffix.** Pre-release Core artifacts use `<ver>.<n>-RC` in the filename (e.g. `pulsar-modules-4.17.0.1-RC.pulsar`). Release drops the suffix. When a Core version graduates from RC to release, every `spa-ui-latest-<core>.properties` for that Core must drop the `-RC` segment too.
- **`spa-ui-latest-latest.properties` is intentionally the most volatile.** It may point at RC Core + develop SPA-UI even while sibling manifests are on stable releases.
- **New tagged SPA-UI release** = (1) add binaries to `pulsar/spa-ui/<ver>/`, (2) create one new `spa-ui-<ver>-<core>.properties` per supported Core, (3) typically also bump `spaui-endorsed` in any `spa-ui-latest-*.properties` whose SPA-UI build is from the matching minor line.
- **New SPA-UI develop build** = drop the `.pulsar` into `pulsar/spa-ui/latest/` and update the `spaui=` line in the relevant `spa-ui-latest-*.properties` files. Commit message convention: `SPA-UI Latest build #<buildno>`.
- **New Core release** = drop binaries into `pulsar/core/<ver>/`, create `pulsar-<ver>.properties`, and update any `spa-ui-latest-<ver>.properties` that should track it. Commit message convention: `Pulsar Core <ver>` (or `Pulsar Core <ver> RC #<n>` during RC).

## Verifying changes

After editing manifests, sanity-check that every referenced file actually exists on disk before committing — a typo or stale path silently breaks consumers:

```bash
# Cross-check every path in a manifest exists under pulsar/
awk -F= '/^(launcher|modules|apps|endorsed|docs|spaui|spaui-endorsed)=/{print $2}' pulsar/<file>.properties \
  | while read p; do [ -e "pulsar/$p" ] || echo "MISSING: $p"; done
```

## Git LFS

`.pulsar` files are LFS-tracked. Cloning fresh requires `git lfs install` once per machine; `git lfs pull` fetches the actual binaries. Don't commit a `.pulsar` without LFS active — you'll push the raw blob into Git history.

## Ant updater (consumer-side, optional)

`scripts/ant/pulsar-update.xml` is shipped here as the reference implementation but is meant to be *copied into a downstream project*. Its `pulsar-update.properties` template has `pulsar.dir`/`devlib.dir` commented out precisely because each consumer fills them in locally. To smoke-test resolution logic from this repo:

```bash
ant -f scripts/ant/pulsar-update.xml -Dpackage=spa-ui -Dversion=<spaui>-<core>
```

(requires `ant-contrib`, bundled at `scripts/ant/lib/ant-contrib-1.0b3.jar`).
