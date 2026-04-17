---
name: build-and-ota
description: Build Thingino firmware and deploy via OTA with safer update choices.
license: MIT
---
# build-and-ota

Use this skill for the normal firmware build/deploy loop when developing or validating camera changes.

## When to use

- You need a standard local build for a specific camera profile.
- You need to deploy to a running camera over the network.
- You want clear guidance on safe OTA target selection.

## Workflow

1. Select camera profile and run build:
   - `CAMERA=<camera_defconfig> make`
   - Fast incremental: `CAMERA=<camera_defconfig> make fast`
2. For package-only iteration:
   - `CAMERA=<camera_defconfig> make rebuild-<package>`
3. Deploy over OTA:
   - Full image including bootloader: `CAMERA=<camera_defconfig> IP=<camera_ip> make ota`
4. Use debug-oriented serial build only when needed:
   - `CAMERA=<camera_defconfig> make dev`

## Notes

- `ota` flashes the full firmware image, including bootloader. Keep power stable and network reliable during flashing.
- If build errors are noisy, `make dev` is easier to read than parallel output.
