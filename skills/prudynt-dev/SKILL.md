---
name: prudynt-dev
description: Develop, debug, and deploy the prudynt streamer on Thingino cameras.
license: MIT
---
# prudynt-dev

Complete onboarding for working on the prudynt-t streamer (Thingino's default video
streamer). Covers local development setup via overrides, rebuild/deploy loop,
config management, diagnostics, and known hardware constraints.

## Architecture overview

prudynt-t is a C++20 video streamer for Ingenic SoC cameras. Key subsystems:

| Subsystem | Source files | Role |
|-----------|-------------|------|
| ISP/Sensor | `IMPSystem.cpp`, `imp_hal.cpp` | Sensor init, ISP tuning, running mode |
| FrameSource | `IMPFramesource.cpp` | Pulls NV12 frames from sensor at configured rate |
| Encoder | `IMPEncoder.cpp` | H264/H265/JPEG hardware encoding |
| VideoWorker | `VideoWorker.cpp` | Polls encoder, writes NALs to msgChannel + taps |
| Audio | `IMPAudio.cpp`, `AACEncoder.cpp`, `Opus.cpp` | Audio capture + encoding |
| RTSP Server | `simple-rtsp/RtspServer.cpp` | RTSP protocol, RTP packetization, TCP/UDP transport |
| Config | `Config.cpp`, `JsonAPI.cpp` | JSON config, runtime API |
| OSD | `OSD.cpp` | Burn-in timestamps, SEI metadata, subtitle track |

Pipeline: Sensor → FrameSource → Encoder → VideoWorker → MsgChannel → RTSP tap → RTP → Client

## Local development setup

### Overrides

prudynt's source is checked out in `overrides/prudynt-t/`. This is a git repo
tracking `origin/stable`. The firmware's Buildroot package (`package/prudynt-t/`)
is configured to rsync from this override directory.

```bash
# Navigate to the override
cd overrides/prudynt-t

# Check current branch and status
git branch
git log --oneline -5
```

The `local.mk` in the firmware root should have `PRUDYNT_T_OVERRIDE_SRCDIR` pointing
to the override. Check with:
```bash
grep prudynt local.mk
```

### Rebuild and deploy

The firmware Makefile provides `rebuild-prudynt-t` which does a full
dirclean+build+reinstall of the prudynt package AND copies the binary to NFS:

```bash
# Build for a specific camera (NFS destination auto-detected from IP)
CAMERA=<camera_name> IP=<camera_ip> make rebuild-prudynt-t
```

Examples:
```bash
CAMERA=cinnado_d1_t31l_sc2336_atbm6031 IP=192.168.88.121 make rebuild-prudynt-t
CAMERA=wyze_cam3_t31x_gc2053_atbm6031   IP=192.168.88.127 make rebuild-prudynt-t
```

The build output goes to:
- Binary: `output/<branch>/<camera>-<kernel>-<libc>-<ip>/per-package/prudynt-t/target/usr/bin/prudynt`
- NFS: `/nfs/prudynt` (host path, accessible from camera at `/mnt/nfs/prudynt`)

Save the full compilation log when rebuilding — errors may scroll by:
```bash
CAMERA=... IP=... make rebuild-prudynt-t 2>&1 | tee /tmp/rebuild-prudynt.log
```

### Deploy to camera

The camera mounts the host's `/nfs` at `/mnt/nfs` via NFS. To deploy:

```bash
# Kill old instance, start new one
ssh root@<camera_ip> 'killall -9 prudynt; sleep 1; /mnt/nfs/prudynt > /tmp/p.log 2>&1 &'

# Verify it started
ssh root@<camera_ip> 'pidof prudynt; grep -E "STREAM PROFILE|fps capped" /tmp/p.log'
```

The binary is also installed to the camera's firmware image during full builds.
For quick iteration, the NFS path avoids reflashing.

## Config management

### Config file

Config is stored in `/etc/prudynt.json` on the camera. This is a JSON file
loaded at startup. Changes require a restart to take effect (except for a few
runtime settings like `running_mode`, `isp_bypass`, and some image controls).

### Runtime API

The `prudyntctl` tool sends JSON to prudynt's HTTP API (`/api/v1/config`).
**Important**: Most settings changed via the API only update the in-memory config
and the JSON file — they do NOT take effect until prudynt restarts. Exceptions:

| Setting | Takes effect immediately? |
|---------|--------------------------|
| `image.running_mode` | Yes (ISP mode change) |
| `image.isp_bypass` | Yes (ISP bypass toggle) |
| `image.brightness`, `image.contrast`, etc. | Yes (ISP controls) |
| `stream0.fps`, `stream0.bitrate`, `stream0.width`, etc. | **No** — restart required |
| `audio.mic_enabled`, `stream0.audio_enabled` | **No** — restart required |
| `osd.sei.enabled`, `osd.burnin.enabled` | **No** — restart required |

```bash
# Read a config value
prudyntctl config get stream0.fps

# Set a config value (writes to file, needs restart for most settings)
prudyntctl json '{"stream0":{"fps":25,"bitrate":3000}}'

# Use jct for direct JSON manipulation
jct /etc/prudynt.json set stream0.fps 25
jct /etc/prudynt.json get stream0.fps
```

### Key config settings

```json
{
  "stream0": {
    "enabled": true,
    "width": 1920, "height": 1080,
    "fps": 0,           // 0 = auto (sensor max), clamped by platform caps
    "bitrate": 3000,    // kbps
    "gop": 30,          // keyframe interval in frames
    "profile": 1,       // 0=Baseline, 1=Main, 2=High
    "mode": "CBR",      // CBR, VBR, CAPPED_VBR, CAPPED_QUALITY
    "audio_enabled": true
  },
  "stream1": {
    "enabled": true,
    "width": 640, "height": 360,
    "fps": 0,
    "bitrate": 1000,
    "profile": 1
  },
  "audio": {
    "mic_enabled": true,
    "mic_format": "OPUS",   // OPUS, AAC, PCM, G711A, G711U
    "mic_hq": false,        // true=48kHz/128kbps, false=16kHz/32kbps
    "spk_enabled": true
  },
  "image": {
    "running_mode": 0,      // 0=day, 1=night
    "isp_bypass": true
  },
  "osd": {
    "sei": { "enabled": true },
    "burnin": { "enabled": true, "format": "%F %T" }
  },
  "rtsp": {
    "port": 554,
    "username": "thingino",
    "password": "thingino",
    "auth_required": true,
    "send_buffer_size": 307200
  }
}
```

## Diagnostics

### Stream analysis with ffprobe

```bash
# Show SDP info
ffprobe rtsp://thingino:thingino@192.168.88.121/ch0

# Count frames in 10 seconds
timeout 10 ffprobe -v error -show_entries frame=pkt_pts_time \
  -select_streams v:0 rtsp://thingino:thingino@192.168.88.121/ch0 | wc -l

# Detailed frame timing and sizes
timeout 25 ffprobe -v error \
  -show_entries frame=pts_time,pkt_size,pict_type \
  -select_streams v:0 \
  -of compact=nk=1 rtsp://thingino:thingino@192.168.88.121/ch0 > /tmp/frames.txt

# Analyze gaps
awk -F'|' '$2 != "N/A" && $2+0 > 0 {
  if (prev>0) { gap=$2-prev; sum+=gap; n++; if(gap>0.05) big++ }
  prev=$2; total++
}
END { printf "frames:%d avg:%.1fms(%.1ffps) gaps>50ms:%d\n",
  total, (sum/n)*1000, n/sum, big }' /tmp/frames.txt
```

### Camera-side checks

```bash
# Check encoder configuration from startup log
grep "STREAM PROFILE" /tmp/p.log

# Check sensor FPS and integration time
cat /proc/jz/isp/isp-m0 | grep -E "OUTPUT FPS|Integration"

# Check available RAM
grep MemTotal /proc/meminfo

# Check kernel warnings
dmesg | grep -i "frame interrupt\|encoder\|sensor"

# Check prudynt log level and recent messages
jct /etc/prudynt.json get general.loglevel
grep -E "WARN|ERROR" /tmp/p.log | tail -20
```

### Binary search for performance regressions

When bisecting to find which feature kills encoder performance:

1. Disable ALL optional features in the config
2. Measure baseline fps at 1080p
3. Re-enable one feature at a time, restart, re-measure
4. The drop reveals the costly feature

```bash
# Quick disable-all
jct /etc/prudynt.json set stream1.enabled false
jct /etc/prudynt.json set stream2.enabled false
jct /etc/prudynt.json set osd.sei.enabled false
jct /etc/prudynt.json set osd.burnin.enabled false
jct /etc/prudynt.json set audio.mic_enabled false
jct /etc/prudynt.json set audio.spk_enabled false
jct /etc/prudynt.json set motion.enabled false
jct /etc/prudynt.json set rtsp.audio_only_enabled false
# Restart and measure
```

### Git bisect for regressions

```bash
cd overrides/prudynt-t
# Create a test branch at a known-good commit
git checkout -b test-bisect <commit_hash>
# Cherry-pick the <unistd.h> fix if needed for GCC 16
git cherry-pick 0bbb467
# Rebuild
cd /home/paul/thingino-builder-image/workspace/firmware
CAMERA=... IP=... make rebuild-prudynt-t
```

## Known hardware constraints

### T31 1080p framerate cap

All T31 variants (T31L, T31X, T31N, T31A) have an encoder hardware limit at
1080p: ~26fps maximum sustained. When configured above 26fps, the encoder
collapses to ~14fps (approximately half). This is NOT a RAM issue — both 36MB
and 93MB devices show identical behavior.

prudynt auto-caps 1080p streams to 25fps on `PLATFORM_T31` builds.
This cap is applied in `IMPSystem.cpp:clamp_stream_to_sensor_limits()`.

### ISP bypass + night mode

Starting the ISP in night mode (`running_mode: 1`) causes encoder pipeline
stalls at 1080p. The init sequence intentionally starts in DAY mode and lets
`daynightd` switch to night via the API after everything is initialized.

### Profile 0 (Baseline) halves framerate

The Ingenic encoder emits frames at **half** the requested rate when H264
profile is set to Baseline (0). Always use Main (1) or High (2) for normal
operation.

## Common pitfalls

1. **API changes don't take effect**: Most `prudyntctl json` settings require a
   restart. Don't debug based on API changes alone — check the startup log.

2. **Wrong sensor driver**: If you've been testing sensor drivers, verify the
   correct one is loaded: `cat /proc/jz/sensor/name`

3. **Config corruption**: Manual JSON edits can introduce syntax errors. Use
   `jct` for programmatic changes. Validate with:
   ```bash
   python3 -c "import json; json.load(open('/etc/prudynt.json'))"
   ```

4. **NFS share not mounted**: If `/mnt/nfs/prudynt` is missing, check NFS:
   ```bash
   mount | grep nfs
   ```

5. **Old binary still running**: `killall -9 prudynt` before starting a new one.

6. **Concurrent streams**: Both ch0 and ch1 share encoder hardware. Disable
   unused streams when debugging ch0 performance.

7. **Stale config values**: Previous tests may have modified the config file.
   Reset to defaults before drawing conclusions.

## Encoder watchdog

prudynt has an encoder watchdog in `VideoWorker.cpp`. After 10 consecutive
`IMP_Encoder_PollingStream` timeouts (~5 seconds at 500ms timeout), it
force-cycles the encoder + framesource:

1. `IMP_Encoder_StopRecvPic`
2. `IMP_FrameSource_DisableChn` / `IMP_FrameSource_EnableChn`
3. `IMP_Encoder_StartRecvPic`
4. `IMP_Encoder_RequestIDR`

This recovers from DMA buffer exhaustion and other pipeline stalls without
requiring a full prudynt restart.

## Related skills

- `nfs-dev-deploy` — generic NFS deployment workflow
- `local-package-overrides` — managing Buildroot package overrides
- `build-and-ota` — full firmware builds and OTA deployment
- `rtsp-stress-test` — RTSP stability and performance testing
