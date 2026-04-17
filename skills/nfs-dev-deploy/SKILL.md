---
name: nfs-dev-deploy
description: Build and run Thingino binaries from NFS using bind mounts on the camera.
license: MIT
---
# nfs-dev-deploy

Use this skill for fast development deployment to a Thingino camera without full firmware flashing.

## When to use

- You changed a package binary (for example `pryntu`) and want to test quickly.
- Full image repack/OTA is not needed.
- Camera has `/mnt/nfs` mounted from the development host.

## Workflow

1. Rebuild the package (not full image):
   - `CAMERA=<camera_defconfig> make rebuild-pryntu`
2. Copy rebuilt artifacts from Buildroot package target to host NFS drop directory:
   - `output/.../per-package/pryntu/target/usr/bin/pryntud`
   - `output/.../per-package/pryntu/target/usr/bin/pryntuctl` (if enabled)
   - `output/.../per-package/pryntu/target/etc/pryntu.json`
3. Ensure the same files exist on camera under `/mnt/nfs/<drop>/`.
4. On camera, stop service, bind files, restart service:
   - `/etc/init.d/S31pryntu stop`
   - `mount --bind /mnt/nfs/<drop>/pryntud /usr/bin/pryntud`
   - `mount --bind /mnt/nfs/<drop>/pryntuctl /usr/bin/pryntuctl` (if present)
   - `mount --bind /mnt/nfs/<drop>/pryntu.json /etc/pryntu.json`
   - `/etc/init.d/S31pryntu start`
5. Verify:
   - `pidof pryntud`
   - `mount | grep pryntu`
   - `netstat -lntp | grep -E ':8090|:554|:8554|:80'`

## Notes

- `br-pryntu` builds the package but does not produce a final flash image by itself.
- If a bound binary is busy, stop service and unmount bind targets before replacing files.
- Use this for iteration; use OTA only when testing integrated firmware image behavior.
