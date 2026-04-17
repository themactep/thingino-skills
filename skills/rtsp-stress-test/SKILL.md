---
name: rtsp-stress-test
description: Run repeatable RTSP playback stress tests with matrix scenarios and log capture.
license: MIT
---
# rtsp-stress-test

Use this skill to measure RTSP stability and compare UDP/TCP behavior with reproducible sessions.

## When to use

- You need evidence for packet loss, decode errors, or delay spikes.
- You want to compare bitrate/GOP/FPS tuning impact.
- You need per-session logs to share with developers.

## Workflow

1. Run baseline (no config changes):
   - `./scripts/rtsp-stress-test.sh --camera <ip>`
2. Run the recommended scenario matrix:
   - `./scripts/rtsp-stress-test.sh --camera <ip> --server-log /mnt/nfs/prudynt.log --recommended-matrix`
3. Compare transport behavior:
   - UDP: `--transport udp`
   - TCP: `--transport tcp`
4. Adjust repetition and depth as needed:
   - `--sessions <n>`
   - `--duration <seconds>`
   - `--pause <seconds>`
   - `--output-dir <dir>`
5. Review scenario summaries for:
   - RTP packet loss
   - `max delay reached`
   - decoder/concealment errors

## Notes

- Host requirements include `ssh`, `ffplay`, `ffmpeg`, `python3`, and `timeout`.
- SSH must be non-interactive for automation.
- Scenario format is `label:bitrate:fps:gop:est_bitrate` (`-` keeps a field unchanged).

