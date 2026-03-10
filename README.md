# HOT4D-C4D2026Plus

An adaptation workspace for bringing **HOT4D** forward toward compatibility with **Cinema 4D 2026 and later**.

## Upstream source

This repository is based on the original HOT4D project:
- Upstream repository: https://github.com/Valkaari/HOT4D
- Original repository owner: Valkaari

HOT4D itself is based on the Houdini Ocean Toolkit and related upstream work referenced by the original project.

## License

This repository remains available under **GNU GPL v2.0**, following the upstream license.

See:
- `LICENSE.txt` for the retained license text
- the upstream repository for original notices and attribution

## Purpose

This repository is intended to:
- preserve the original HOT4D codebase
- investigate build and SDK breakages against newer Cinema 4D versions
- implement compatibility changes for Cinema 4D 2026 and later
- publish adaptation work under the same open-source license family required by upstream

## Repository layout

The code structure is kept close to upstream:
- `source/` — plugin and ocean simulation source code
- `res/` — Cinema 4D resources, descriptions, strings, icons
- `project/` — Cinema 4D module build metadata
- `scripts/` — local helper scripts for SDK-backed bring-up

## Status

Initial fork-style bootstrap is complete.
Cinema 4D 2026 and later compatibility analysis and porting work is in progress.

The first real SDK-backed integration pass has now been exercised:
- the correct integration route is the SDK's **top-level CMake** plus `MAXON_SDK_CUSTOM_PATHS_FILE`
- this repo is discovered correctly as an external module by the SDK CMake flow
- the currently provided SDK payload is incomplete (missing the `frameworks/` tree), so configure stops before HOT4D source compilation begins

Current notes:
- see `BUILDING.md` for the verified SDK workflow and current blocker
- see `TODO-C4D2026Plus.md` for the active porting checklist
- see `docs/MIGRATION-NOTES-C4D2026Plus.md` for the source-tree responsibility/risk map

## Attribution

This repository is an adaptation/maintenance continuation and is **not** the original upstream project.
If you need original history and context, start here:
https://github.com/Valkaari/HOT4D
