---
name: device-smoke-cycle
description: Build a library/binary locally, stage it on a Thingino camera, run a consumer against it, capture dmesg/logread/logcat, then reboot.
license: MIT
---
# device-smoke-cycle

Use this skill for a fast iteration loop: build something locally, stage it on the camera, run a consumer that links/executes against the staged copy, collect logs, reboot. Each project wires the specifics; the cycle shape is the same.

## When to use

- You changed a library (e.g. `libimp.so`, `libsysutils.so`) and want to see how a consumer (e.g. Raptor's `S31raptor`) behaves against the new build.
- You want a clean dmesg + logread (+ logcat) bundle from a freshly-started consumer.
- You want the device returned to a known state (rebooted) before the next iteration.
- You want agent-friendly output: a small, comparable log bundle per cycle.

## Inputs

Provide via env vars, CLI args, or a `## device-smoke-cycle` block in the project's `CLAUDE.md`. `CLAUDE.md` takes precedence.

- `BUILD_CMD`   - build invocation. Probe order if unset: `./build-for-device.sh` → `./build.sh` → `./script.sh` → `make`.
- `ARTIFACTS`   - space-separated host paths to stage (e.g. `lib/libimp.so lib/libsysutils.so`).
- `STAGE_DIR`   - device directory to `scp` into. Default: `/opt`.
- `START_CMD`   - command run on device that consumes the staged artifacts. Typically prefixed with `LD_LIBRARY_PATH=$STAGE_DIR` so the staged libs win over `/usr/lib`.
- `SETTLE_SECS` - seconds to wait after start before capturing. Default: `10`.
- `COLLECT`     - commands whose output to capture. Default: `dmesg logread logcat`.
- `POST_ACTION` - action after capture. Default: `reboot`.
- `IP`          - camera IP (required).
- `SSH_USER`    - default `root`.

## Workflow

1. Resolve the build command:
   - Use `BUILD_CMD` if set (env, arg, or `CLAUDE.md`).
   - Otherwise probe the repo root in order: `build-for-device.sh`, `build.sh`, `script.sh`, then `make`.
   - Refuse to guess if none match - ask for `BUILD_CMD` explicitly.

2. Build locally:
   - Run `BUILD_CMD` (pass platform/flags as the project expects, e.g. `./build-for-device.sh T31`).

3. Stage `ARTIFACTS` under `STAGE_DIR` on the device:
   - `scp $ARTIFACTS "$SSH_USER@$IP:$STAGE_DIR/"`
   - If the build script itself uploads, skip this step.

4. Start the consumer against the staged copy and show initial syslog:
   - `ssh "$SSH_USER@$IP" "$START_CMD & logread"`
   - Typical `START_CMD`: `LD_LIBRARY_PATH=/opt /opt/S31raptor start`.

5. Wait `SETTLE_SECS`, then capture each `COLLECT` command's output to a timestamped file:
   - `TS=$(date -u +%Y%m%dT%H%M%SZ)`
   - For each `cmd` in `COLLECT`:
     - `ssh "$SSH_USER@$IP" "sleep $SETTLE_SECS; $cmd 2>/dev/null" > "logs/$TS-$cmd.log"` (only sleep on the first one)
   - Or collapse into one SSH round-trip with clear `===CMD===` banners between sections.

6. Run `POST_ACTION` to reset state:
   - Default: `ssh "$SSH_USER@$IP" 'reboot'` (use `reboot -f` only if hung).

7. Hand the captured log files back to the agent for higher-level analysis and to plan the next change.

## Project wiring via CLAUDE.md

Preferred: add a pinned block so the cycle is explicit and greppable. Example for openimp → Raptor:

```markdown
## device-smoke-cycle
BUILD_CMD=./build-for-device.sh T31
ARTIFACTS=lib/libimp.so lib/libsysutils.so
STAGE_DIR=/opt
START_CMD=LD_LIBRARY_PATH=/opt /opt/S31raptor start
```

The skill reads this block first, env/CLI second, probed defaults last.

## Notes

- `LD_LIBRARY_PATH=$STAGE_DIR` is the load-bearing trick: it lets the staged libs win without overwriting `/usr/lib/` or unpacking a new firmware image.
- If the consumer's init script launches daemons from `/usr/bin/`, `LD_LIBRARY_PATH` still applies (inherited through `start-stop-daemon`), but the daemon *binaries* themselves come from `/usr/bin/`. To test modified daemon binaries, either bind-mount `$STAGE_DIR/<daemon>` over `/usr/bin/<daemon>` first, or run the daemons directly instead of via the init script.
- Stop any already-running instance of the consumer before starting the staged copy, or the two will race.
- `logcat` is not present on every Thingino build - redirect stderr and do not fail the cycle if it is missing.
- Keep a cycle under ~60 seconds wall-clock so successive runs produce comparable logs.
- SSH must be non-interactive (key-based auth) for agent-driven runs.
- A reboot after each cycle guarantees `/dev/shm/rss_ring_*`, stale daemons, and any mount tricks are cleared - do not skip it when iterating.
