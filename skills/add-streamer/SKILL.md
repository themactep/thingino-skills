---
name: add-streamer
description: Add and correctly plumb a new streamer backend in Thingino.
license: MIT
---
# add-streamer

Use this skill when you need to add a new streamer backend and wire it cleanly into Thingino's Buildroot selection flow.

## When to use

- You are introducing a new streamer package (for example, `valisa`).
- You need the backend to appear as a selectable streamer option.
- You need defaults and exported `STREAMER` value to stay consistent across SoC families.

## Required plumbing checklist

1. Add a choice symbol in `package/thingino-streamer/Config.in`:
   - `config BR2_PACKAGE_THINGINO_STREAMER_<NAME>`
   - `select BR2_PACKAGE_<PKG>`
   - Gate with `depends on !BR2_SOC_FAMILY = "a1"` if `a1` must not run a streamer.
2. Ensure the package `Config.in` is tied to the choice symbol:
   - `depends on BR2_PACKAGE_THINGINO_STREAMER_<NAME>`
3. Source the package `Config.in` from the streamer block:
   - `if BR2_PACKAGE_THINGINO_STREAMER`
   - `source "$BR2_EXTERNAL_THINGINO_PATH/package/<pkg>/Config.in"`
4. Avoid exposing streamer packages outside the streamer selection flow.
5. Update `thingino.mk` `STREAMER` mapping:
   - map package enablement to a stable runtime name (for example, `STREAMER := valisa`)
6. Keep streamer defaults explicit in `package/thingino-streamer/Config.in`:
   - `none` for `BR2_THINGINO_TOOLCHAIN`
   - `none` for `BR2_SOC_FAMILY = "a1"` (if `a1` has no streamer)
   - desired default streamer for remaining targets

## Workflow

1. Edit `package/thingino-streamer/Config.in` first:
   - add option symbol, help text, dependencies, and default rules.
2. Edit `package/<pkg>/Config.in`:
   - add dependency on `BR2_PACKAGE_THINGINO_STREAMER_<NAME>`.
3. Ensure `source .../package/<pkg>/Config.in` lives under the streamer block in `package/thingino-streamer/Config.in`.
4. Remove duplicate top-level sourcing from root `Config.in` if present.
5. Edit `thingino.mk`:
   - add explicit `ifeq` branch mapping package symbol to `STREAMER` value.
   - keep fallback deterministic (`none` if no streamer selected).

## Validation

1. Non-`a1` profile:
   - `CAMERA=<non_a1_camera> make defconfig`
   - verify `.config` streamer symbols match expected default.
2. `a1` profile:
   - `CAMERA=<a1_camera> make defconfig`
   - verify `BR2_PACKAGE_THINGINO_STREAMER_NONE=y`.
3. Confirm no unintended symbol exposure:
   - streamer package symbol should follow the streamer choice logic.

## Common pitfalls

- Package `Config.in` sourced at top level, bypassing streamer choice.
- Missing `depends on !BR2_SOC_FAMILY = "a1"` on streamer options that should exclude `a1`.
- Default still pointing to an old backend after introducing a new one.
- `thingino.mk` missing a branch for the new package, causing wrong exported `STREAMER`.
