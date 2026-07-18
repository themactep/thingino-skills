# thingino-skills

Thingino-specific Copilot skills for firmware development and deployment workflows.

## Skills

- `nfs-dev-deploy`: Build `pryntu`, stage artifacts on NFS, bind-mount them on the camera, and restart services for fast iteration.
- `local-package-overrides`: Create and manage Buildroot package source overrides with `scripts/manage-package-overrides.sh`.
- `rtsp-stress-test`: Run reproducible RTSP UDP/TCP stress matrices and collect per-session logs.
- `collect-diagnostics`: Gather and share Thingino diagnostics from Web UI, shell, or SD-card trigger.
- `build-and-ota`: Use the standard Thingino build + OTA workflow safely for day-to-day firmware updates.
- `add-streamer`: Add and correctly plumb a new streamer backend across Kconfig and build wiring.
- `thingino-product-styling`: Apply Thingino-consistent visual styling for product pages and camera cards.
- `device-smoke-cycle`: Build locally, stage to `/opt`, run a consumer (e.g. `S31raptor`) against the staged copy, collect dmesg/logread/logcat, then reboot.
- `worktree-parallel-dev`: Run parallel tasks/agents in isolated git worktrees with correct buildroot submodule, patch, and dl-cache setup (`scripts/worktree.sh`).

## Repository layout

```text
skills/<skill-name>/SKILL.md
```

Each skill follows the Copilot skill format with YAML front matter and Markdown instructions.
