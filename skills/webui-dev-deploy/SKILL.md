---
name: webui-dev-deploy
description: Deploy Thingino WebUI changes (HTML/CSS/JS/CGI) straight to a camera without rebuilding firmware or flashing.
license: MIT
---
# webui-dev-deploy

Iterate on the Thingino Web UI on a live camera without a full build or OTA.

## When to use

- You changed files under `package/thingino-webui/files/www/` or
  `package/prudynt-t/files/www/`: HTML, CSS, JS, or CGI shell scripts.
- You want to see a UI change on the camera in seconds, not after an 11-minute
  rebuild and a reflash.
- The change is a static or interpreted file, not a compiled binary.

For binaries (`prudynt`, kernel, drivers) use `nfs-dev-deploy`, a package
rebuild, or OTA instead.

## Where the files land

| Repo path | Camera path |
|-----------|-------------|
| `package/thingino-webui/files/www/...` | `/var/www/...` |
| `package/prudynt-t/files/www/...` | `/var/www/...` |

Both packages install into the same `/var/www` tree. Subdirectory `a/` holds
the JS/CSS, `x/` holds the CGI scripts (usually `#!/bin/sh`).

## Upload

The camera runs dropbear, so `scp` needs `-O`:

```sh
scp -O -o StrictHostKeyChecking=no \
  package/thingino-webui/files/www/a/main.js \
  root@<camera>:/var/www/a/
```

`/var/www` lives on the writable overlay (`/dev/mtdblockN` mounted at
`/overlay`, overlayfs upper). Uploads persist across reboots and survive until
the next firmware flash.

No service restart is needed: uhttpd serves from disk.

## Verify

```sh
ssh root@<camera> 'grep -n "<marker>" /var/www/a/main.js'
```

## Gotchas

- The build injects an asset tag (`?v=<mtime>`) into HTML asset URLs via
  `scripts/apply_asset_tag.py`. Uploaded HTML carries no tag, so a browser may
  keep a cached script; hard-refresh (Ctrl-Shift-R) after uploading.
- Paranoid mode and CDN-fallback rewrites also happen at build time. On a
  camera built with them, either upload the built file from the target tree
  (`output/<...>/target/var/www/...`) or preserve the rewrites.
- This only changes the running device. It does not change the image, so land
  the change in git and rebuild before it ships.
