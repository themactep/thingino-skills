---
name: webui-plugin
description: Create a Thingino WebUI plugin that adds pages, navigation items, and scripts to the camera web interface when an optional package is installed.
license: MIT
---
# webui-plugin

Use this skill when you need to add web UI pages, navigation items, scripts, or CGI endpoints that only appear when a specific optional package is installed.

## When to use

- You are creating or modifying a Buildroot package that needs web UI configuration pages (e.g. motors, doorbell, daynightd, go2rtc, nino).
- You want the web UI to show a new Settings/Tools/Services menu item only when the package is enabled in the build config.
- You need to inject scripts or HTML into the preview page, or load a global script on every page.
- You are migrating an existing package's web UI integration away from hardcoded core webui logic.

## How it works

The plugin system is **build-time only** — no runtime scanning overhead. Each plugin ships a JSON manifest (`<name>.webui.json`) and its own web files (HTML, JS, CGI) inside its package directory. A Python assembly script (`assemble_plugins.py`) runs as a Buildroot `TARGET_FINALIZE_HOOKS` after all packages are installed. It:

1. Scans `$(TARGET_DIR)/var/www/a/plugins/*.webui.json`
2. Validates for duplicate names / page conflicts
3. Generates `/var/www/a/plugins.js` with merged nav config and feature flags
4. Injects plugin scripts/styles into HTML pages
5. Injects preview HTML/scripts into `preview.html` / `preview-raptor.html`

At runtime, `navigation.js` reads `window.thinginoUIConfig.plugins` and splices the declared items into the navigation menu with positional control.

## Plugin manifest format

Create a file `package/<name>/files/<name>.webui.json`:

```jsonc
{
  // Required. Must match filename and be lowercase alphanumeric with
  // hyphens/underscores only.
  "name": "myplugin",

  // Optional human-readable label for logging.
  "label": "My Plugin",

  // Optional API version (currently 1).
  "apiVersion": 1,

  // ── Navigation contributions ────────────────────────────────
  "nav": [
    {
      // Target section.  Standard sections:
      //   ddSettings, ddTools, ddServices, ddStreamer, ddInfo, ddHelp
      "section": "ddSettings",

      // Where to insert.  Values:
      //   "append" (default), "prepend",
      //   "after:<label>", "before:<label>", "index:<n>"
      "position": "after:GPIO pins",

      // One or more items.  Each needs label and href.
      "items": [
        { "label": "My Config", "href": "/config-myplugin.html" }
      ]
    }
  ],

  // ── Global scripts (loaded on EVERY page) ───────────────────
  // Keep these small — they run on all pages including preview.
  "scripts": [
    "/a/myplugin-banner.js"
  ],

  // ── Global stylesheets (rarely needed) ──────────────────────
  "styles": [],

  // ── Preview page injection ──────────────────────────────────
  "preview": {
    // Scripts loaded only on preview.html and preview-raptor.html.
    "scripts": [
      "/a/preview-myplugin.js"
    ],
    // HTML snippet injected at <!-- THINGINO_PLUGIN_PREVIEW_BODY -->
    // marker in preview pages.  Use null if not needed.
    "html": "<div id=\"myplugin-overlay\" style=\"display:none\">...</div>"
  },

  // ── Feature flags ───────────────────────────────────────────
  // Merged into thinginoUIConfig.device at build time.
  // Accessible to all JS as uiConfig.device.<key>.
  "featureFlags": {
    "myplugin": true
  },

  // ── Declared pages (for conflict detection) ─────────────────
  "pages": [
    "/config-myplugin.html"
  ],

  // ── Declared CGI endpoints (for conflict detection) ─────────
  "cgi": [
    "/x/json-myplugin.cgi",
    "/x/json-myplugin-status.cgi"
  ]
}
```

## Package wiring checklist

1. **Place web files** in your package directory:
   ```
   package/<name>/files/
   ├── <name>.webui.json          ← manifest
   └── www/
       ├── config-<name>.html     ← HTML page(s)
       ├── a/
       │   ├── config-<name>.js   ← page-specific JS
       │   └── preview-<name>.js  ← preview script (if needed)
       └── x/
           ├── json-<name>.cgi    ← CGI endpoint(s)
           └── ...
   ```

2. **Add dependency** and conditionally define web install commands.
   The `INSTALL_WWW_CMDS` variable is only defined when the webui is
   built, then referenced inside the main install define.  This avoids
   `ifeq` inside a define block (which would be passed to the shell).

   ```make
   ifeq ($(BR2_PACKAGE_THINGINO_WEBUI),y)
   MYPLUGIN_DEPENDENCIES += thingino-webui
   define MYPLUGIN_INSTALL_WWW_CMDS
       $(INSTALL) -d $(TARGET_DIR)/var/www/a
       $(INSTALL) -d $(TARGET_DIR)/var/www/x
       $(INSTALL) -d $(TARGET_DIR)/var/www/a/plugins
       $(INSTALL) -D -m 0644 $(@D)/files/www/config-myplugin.html \
           $(TARGET_DIR)/var/www/config-myplugin.html
       $(INSTALL) -D -m 0644 $(@D)/files/www/a/config-myplugin.js \
           $(TARGET_DIR)/var/www/a/config-myplugin.js
       $(INSTALL) -D -m 0755 $(@D)/files/www/x/json-myplugin.cgi \
           $(TARGET_DIR)/var/www/x/json-myplugin.cgi
       $(INSTALL) -D -m 0644 $(@D)/files/myplugin.webui.json \
           $(TARGET_DIR)/var/www/a/plugins/myplugin.webui.json
   endef
   endif
   ```

3. **Reference it** inside your main install define:
   ```make
   define MYPLUGIN_INSTALL_TARGET_CMDS
       # ... binary installs ...
       $(MYPLUGIN_INSTALL_WWW_CMDS)
   endef
   ```

   **Important:** Always add `$(INSTALL) -d` for `/var/www/a`, `/var/www/x`,
   and `/var/www/a/plugins` before installing files there. Per-package
   Buildroot builds use isolated target directories that don't inherit
   `/var/www/` from `thingino-webui`.

4. **Validate** before building:
   ```bash
   scripts/check-plugins.sh
   ```
   This collects all `*.webui.json` manifests and runs the assembly
   validator in `--check-only` mode.  Does not require `CAMERA=`.

5. **Build and verify** — the assembly pipeline handles nav integration,
   script injection, preview injection, conflict detection, and asset
   tag management automatically.

## If you are migrating an existing package

Checklist for pulling an existing web UI integration out of core webui:

1. **Create the manifest** (`<name>.webui.json`) following the schema above.
2. **Move web files** from `package/thingino-webui/files/www/` to your
   package's `files/www/`.
3. **Remove from `navigation.js`** — delete any hardcoded nav items in
   `buildDefaultMenu()` that belong to this plugin.
4. **Remove from `thingino-webui.mk`** — delete the `$(INSTALL)` lines
   for the moved files.  If they were gated by a `BR2_PACKAGE_*` check,
   remove the entire conditional block.
5. **Remove runtime probing from `S48webui-config`** — if the package had
   a runtime feature-flag probe (e.g. checking for a CGI file or
   `thingino.json` key), delete it.  Feature flags now come from the
   manifest.
6. **Add `DEPENDENCIES += thingino-webui`** to your package's `.mk`.

## Reference: existing plugins

| Plugin | Package | Manifest location |
|--------|---------|-------------------|
| motors | `thingino-motors` | `package/thingino-motors/files/motors.webui.json` |
| doorbell | `wyze-accessory` | `package/wyze-accessory/files/doorbell.webui.json` |
| daynightd | `thingino-daynightd` | `package/thingino-daynightd/files/daynightd.webui.json` |

## Validation

After completing the wiring:

```bash
scripts/check-plugins.sh            # validates all manifests
CAMERA=<camera> make                # full build test
```

Check the generated output on the camera:
- `/var/www/a/plugins.js` should contain your plugin's nav and feature flags.
- Your pages should appear in the navigation menu at the declared position.
- Feature flags should be accessible as `uiConfig.device.<key>` in JS.
