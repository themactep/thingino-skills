---
name: uboot-patch-workflow
description: Create and apply follow-up U-Boot patches in thingino-firmware without modifying the large 0001 patch
source: auto-skill
extracted_at: '2026-06-03T06:52:58.263Z'
---

# U-Boot Patch Workflow (thingino-firmware)

## Rule

**Never modify the large `0001-from-2026.04-to-thingino.patch`** (~12 MB). Work with the patched sources in the build output and create numbered follow-up patches (`0002-`, `0003-`, etc.).

## Procedure

### 1. Find the patched source

```bash
# Build output dir after patches are applied:
output/<branch>/<camera>-<kernel>-<libc>/build/uboot-2026.04/
```

### 2. Edit and generate the patch

```bash
cd /tmp
cp <build>/path/to/file.c file.orig
cp file.orig file.new
# edit file.new with your changes
diff -u file.orig file.new
```

### 3. Wrap as a proper patch

Create `package/all-patches/uboot/2026.04/000N-short-desc.patch`:

```
From: Your Name <email>
Date: ...
Subject: [PATCH] short description

Commit message explaining what and why.

Signed-off-by: ...
---
 path/to/file.c | 1 +
 1 file changed, 1 insertion(+)

diff --git a/path/to/file.c b/path/to/file.c
--- a/path/to/file.c
+++ b/path/to/file.c
@@ -line,context +line,context @@
  context line
+added line
  context line
```

### 4. Verify and rebuild

```bash
cd <build>/uboot-2026.04
patch --dry-run -p1 < <repo>/package/all-patches/uboot/2026.04/000N-*.patch

# If clean:
make rebuild-uboot
```

### 5. Clean up

```bash
rm -f /tmp/file.orig /tmp/file.new /tmp/*.diff
```

## Why this approach

The `0001` patch is a monolithic patch generated from the thingino U-Boot repo on top of the vanilla U-Boot 2026.04 release. Editing it directly is error-prone (wrong line numbers, corrupted index lines, 10MB+ file). Follow-up patches are cleaner, easier to review, and follow Buildroot's standard patch application order (sorted numerically).
