---
name: collect-diagnostics
description: Generate and share Thingino diagnostics from UI, shell, or SD-card trigger.
license: MIT
---
# collect-diagnostics

Use this skill to gather support-ready diagnostics from a Thingino camera.

## When to use

- The camera has instability, connectivity, or service issues.
- You need a shareable report URL for support/debugging.
- The camera has no internet and you need local report output.

## Workflow

1. Preferred UI path:
   - Open **Information -> Share Diagnostics Info** in Web UI.
2. Shell method:
   - Local temp file (default): `thingino-diag`
   - Upload to server: `thingino-diag -u`
   - Upload and return JSON link: `thingino-diag -j`
3. Offline/file mode:
   - `thingino-diag -o /path/file`
   - This writes a local diagnostics file.
4. No shell access fallback:
   - Put an empty `.diag` file in the root of a blank SD card.
   - Insert card into running camera to trigger report generation on the card.

## Notes

- For RTSP-specific reliability investigations, pair this with `rtsp-stress-test`.
- Legacy local-save flag `-l [path]` is still accepted, but prefer `-o`.
