---
name: local-package-overrides
description: Set up and iterate on Thingino package source overrides using local clones.
license: MIT
---
# local-package-overrides

Use this skill when you need to modify a package source locally without editing Buildroot internals.

## When to use

- You need fast iteration on a package from local source.
- You want per-repo, per-camera, or per-device override scoping.
- You need to disable/re-enable/remove overrides cleanly.

## Workflow

1. List packages or filter candidates:
   - `./scripts/manage-package-overrides.sh -l`
   - `./scripts/manage-package-overrides.sh -l thingino-*`
2. Create an override for a package:
   - `./scripts/manage-package-overrides.sh <package-name>`
   - Auto mode: `./scripts/manage-package-overrides.sh -a <pattern>`
3. Edit local source in `overrides/<package>/`.
4. Rebuild only that package:
   - `make <package>-rebuild`
   - Optionally rebuild image too: `make <package>-rebuild all`
5. Manage existing overrides:
   - Disable: `./scripts/manage-package-overrides.sh -d <package>`
   - Enable: `./scripts/manage-package-overrides.sh -e <package>`
   - Update: `./scripts/manage-package-overrides.sh -u <package>`
   - Remove mapping: `./scripts/manage-package-overrides.sh -r <package>`

## Notes

- Override mappings can live in repository root `local.mk` or user-scoped files:
  - `user/common/local.mk`
  - `user/<camera>/local.mk`
  - `user/<camera>/<ip>/local.mk`
- Buildroot copies override sources with `rsync`, so local edits are picked up by package rebuilds.

