---
name: open-sensor-drivers
description: Work on Thingino's T31 open-ISP mainline V4L2 sensor drivers. Use when porting a sensor, wiring its DT fragment and camera config, adding the per-sensor AE gain LUT in the open-tx-isp bridge, or debugging AE breathing and gain-range problems on the open ISP path.
license: MIT
---
# open-sensor-drivers

Use this skill for the T31 open-ISP sensor path on Linux 7.2.x: porting a
mainline V4L2 sensor, wiring its device-tree fragment and camera config, or
fixing the per-sensor AE gain model in the open-tx-isp V4L2 bridge.

The detailed reference is `docs/media/t31-mainline-sensor-drivers.md` in the
firmware repo. Read it before changing a sensor.

## Architecture

| Part | Location | Responsibility |
| --- | --- | --- |
| Mainline V4L2 driver | `linux/drivers/media/i2c/<sensor>.c`, added as a patch in `package/all-patches/linux/7.2.11/` | Register sequences, V4L2 controls, index-to-register gain table |
| V4L2 bridge | open-tx-isp `driver/t31/tx_isp_v4l2_sensor.c` | Implements the Ingenic vendor ABI over the V4L2 subdev and owns the per-sensor Q10 gain model |
| ISP core | open-tx-isp `driver/t31/tx_isp_module.c`, `tx_isp_core.c` | AE, CSI/ISP pipeline, frame production |
| DT fragment | `configs/sensors/t31/<model>.dtsi` | I2C address, supplies, clock, MIPI endpoint; injected before the camera fragment |
| Camera selection | `configs/cameras/<camera>/<camera>_defconfig` | `BR2_SENSOR_1_NAME="<model>"` drives `SENSOR_1_MODEL`, `CONFIG_VIDEO_<MODEL>`, `SENSOR_DTSI_FILE` |
| Raptor config | `package/thingino-raptor/thingino-raptor-conf.mk` | Seeds `[sensor] name`/`i2c_addr` when the explicit raptor options are empty |
| Startup gate | `package/thingino-raptor/files/S30raptor-sensor-wait` | Holds the streamer until the mainline sensor's I2C driver is bound |

The open-tx-isp source is fetched by `package/open-tx-isp/open-tx-isp.mk`.
Change the bridge with numbered patches in `package/open-tx-isp/`; do not
edit the fetched tree.

## Gain-model contract

The bridge attaches to a probed V4L2 subdev and requires:

- `V4L2_CID_EXPOSURE`
- `V4L2_CID_ANALOGUE_GAIN`
- `V4L2_CID_HBLANK`
- `V4L2_CID_VBLANK`
- `V4L2_CID_LINK_FREQ`
- `V4L2_CID_PIXEL_RATE`

`V4L2_CID_ANALOGUE_GAIN` is an index. The Ingenic AE needs a **Q10 linear
gain** (`1024` = 1x, `2048` = 2x, `32768` = 32x), so the bridge maps the
index through its per-sensor LUT. `attr->max_dgain` is 0; exposure is
integration time plus analog gain.

The vendor SDK's `sensor_again_lut` second column is **log2(gain) in Q16**,
not a linear gain. Convert each row with:

```python
q10 = round(1024 * 2 ** (gain_log2_q16 / 65536))
```

Do **not** use `1 + coarse + (fine - 128) / 128`. It is only valid for
coarse 0 and reports 9x, 10x and 12x where the sensor gives 2x, 4x and 8x.
That mistake creates a fake gain gap and makes the AE "breathe".

Current models in `tx_v4l2_gain_models[]`:

| Sensor | Entries | Max index | Max gain |
| --- | ---: | ---: | ---: |
| `sc2336` | 161 | 160 | 32x |
| `sc2336p` | 161 | 160 | 32x |
| `sc4236` | 63 | 62 | 15x |
| `gc2083` | 29 | 28 | 125x |
| `gc4023` | 25 | 24 | 64x |

The bridge caps the index at the sensor's `V4L2_CID_ANALOGUE_GAIN.maximum`
and sets `attr->max_again` to the LUT value at that index. The LUT size
should be the control maximum + 1.

## Adding a sensor

1. Port the mainline driver as a patch in
   `package/all-patches/linux/7.2.11/`. Keep the register sequences, the
   V4L2 controls, and a `*_gains[]` table. Put the **log2 Q16 gain value**
   next to each row (comment or field); the LUT cannot be derived without it.
2. Add the Kconfig symbol and Makefile entry in
   `0099-media-i2c-enable-thingino-t31-sensors.patch`. The symbol must be
   `VIDEO_<MODEL>` in upper case: `open-tx-isp.mk` builds
   `CONFIG_VIDEO_$(SENSOR_1_MODEL uppercase)`. A mismatch silently drops the
   driver from the kernel.
3. Add `configs/sensors/t31/<model>.dtsi` with the `compatible` string,
   `reg`, supplies, clock, reset/power GPIOs and the MIPI endpoint. The
   `THINGINO_KOPT_SYNC_SENSOR_DTSI` hook injects it before the camera
   fragment.
4. Set `BR2_SENSOR_1_NAME="<model>"` in the camera defconfig. This drives the
   kernel config, the DT fragment path, and the raptor `[sensor]` fallback.
   Explicit `BR2_PACKAGE_THINGINO_RAPTOR_CONF_SENSOR_NAME` /
   `..._I2C_ADDR` still win.
5. Generate the Q10 LUT and add it to the bridge:

   ```python
   import re

   src = open('drivers/media/i2c/sc4236.c').read()
   table = re.search(r'sc4236_gains\[\] = \{(.*?)\n\};', src, re.S).group(1)
   rows = re.findall(r'/\*\s*(0x[0-9a-fA-F]+|\d+)\s*\*/', table)
   vals = [round(1024 * 2 ** (int(x, 0) / 65536)) for x in rows]
   print(', '.join(str(v) for v in vals))
   ```

   Add `tx_v4l2_<model>_gain_lut[]` and a `{ "<model>", ... }` entry in
   `tx_v4l2_gain_models[]`, carried as the next `package/open-tx-isp/`
   patch. The name must match the sensor model string exactly: it is what
   `rvd` passes to `IMP_ISP_AddSensor` and what the bridge compares against
   the V4L2 client name.
6. Build and validate.

## Validation

```bash
CAMERA=<camera> make fast
```

On the camera:

```bash
dmesg | grep tx-isp-v4l2
cat /proc/jz/isp/isp-m0
```

Expect:

```text
tx-isp-v4l2: matched <model> at 0x..
tx-isp-v4l2: gain model <model> (N entries, max index M)
```

In `/proc/jz/isp/isp-m0`, check `SENSOR NAME`, `SENSOR Integration Time`,
`SENSOR analog gain`, `open-tx-isp attr analog gain`,
`open-tx-isp attr max analog gain`, and `ISP digital gain`.

AE breathing test (20 s, stable scene):

```bash
ffmpeg -rtsp_transport tcp -i 'rtsp://<user>:<pass>@<camera>:554/ch0' \
  -t 20 -vf 'signalstats,metadata=print:file=/tmp/yavg.txt' -an -f null -
awk -F= '/YAVG=/{print $2}' /tmp/yavg.txt > /tmp/yavg.vals
```

A stable scene should stay within a few luma levels over 20 s. The SC2336
went from a `74..151` range (breathing) to `116.15..116.50` once the LUT was
correct. If the gain index alternates across a large LUT step, the table is
wrong.

Also check low light and the day/night transition: the AE must walk the
whole index range without a jump.

## Common pitfalls

- Deriving gain from the coarse/fine register formula instead of the SDK
  log2-Q16 column. This is the classic breathing bug.
- LUT size not matching the V4L2 control maximum, or the model name not
  matching the driver/subdev name.
- Kconfig symbol not `VIDEO_<MODEL>` uppercase.
- Adding the driver without the DT fragment or camera `BR2_SENSOR_1_NAME`.
- Forgetting the log2 Q16 gain column in a new mainline port; without it the
  LUT cannot be generated.
- Relying on raptor `[image] max_again`/`max_dgain` to cap the open path.
  The bridge uses `attr->max_again` from the LUT and the V4L2 control max.
- Adding a model but not carrying it as a `package/open-tx-isp/` patch; the
  fetched source is not the place to edit.
- The bridge is single-sensor (`tx_isp_v4l2_sensor_sd`); multi-sensor on the
  mainline path is not implemented.

## File map

```text
docs/media/t31-mainline-sensor-drivers.md
package/all-patches/linux/7.2.11/0020-media-i2c-add-sc2336.patch
package/all-patches/linux/7.2.11/0099-media-i2c-enable-thingino-t31-sensors.patch
configs/sensors/t31/<model>.dtsi
configs/cameras/<camera>/<camera>.dtsi
configs/cameras/<camera>/<camera>_defconfig
package/open-tx-isp/0017-t31-select-per-sensor-gain-model.patch
package/thingino-raptor/thingino-raptor-conf.mk
package/thingino-raptor/files/S30raptor-sensor-wait
linux/linux-ext-thingino-kopt.mk
package/thingino-kopt/inject-camera-dtsi.sh
```
