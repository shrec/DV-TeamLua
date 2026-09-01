# MUPF Plugins

This page is the authoritative external-developer contract for plugin layout, manifest fields, capabilities, limits, reload behavior, and deployment.

> Audited against the current `dev` implementation on **2026-08-31**. Fields that are accepted as metadata but do not affect runtime behavior are marked explicitly.

## Plugin layout

A server-backed plugin is developed as a source folder under `Data/ClientLuaPlugin/`:

```text
Data/ClientLuaPlugin/
├─ policy.json
└─ com.example.ranking/
   ├─ plugin.json
   ├─ server/
   │  └─ server.lua
   └─ client/
      ├─ index.html
      ├─ launcher.html       # optional
      ├─ style.css
      └─ images/*
```

- `server.lua` runs only in the GameServer.
- `plugin.json` and `client/*` are packed and streamed to the player on connect.
- The client keeps received files in a per-plugin in-memory virtual file system.
- There is no client-side Lua runtime. Client logic is JavaScript inside the HTML entry page.

The main page path is currently fixed to `client/index.html`. If a launcher is configured, its page path is fixed to `client/launcher.html`.

## Minimal manifest

```json
{
  "id": "com.example.ranking",
  "name": "Server Ranking",
  "version": "1.0.0",
  "apiVersion": 1,
  "minApiVersion": 1,

  "server": {
    "entry": "server/server.lua"
  },

  "client": {
    "window": {
      "width": 640,
      "height": 560,
      "title": "Server Ranking",
      "opacity": 235
    }
  },

  "entryPoints": {
    "hotkey": "F5",
    "uiButton": {
      "label": "Ranking"
    }
  },

  "permissions": ["db.query", "player.readBasic"]
}
```

### Runtime fields

| Field | Current behavior |
|---|---|
| `id` | Required unique string. It is the policy key and must exactly match `PluginRegister(id, hooks)`. |
| `name` | Display metadata; defaults to `id`. |
| `version` | Plugin metadata; defaults to `0.0.0`. |
| `apiVersion` | API version targeted by the plugin; defaults to `1`. |
| `minApiVersion` | Minimum client host API. A newer requirement quarantines the plugin instead of running it incorrectly. |
| `server.entry` | Relative source-folder path to server Lua. Use `server/server.lua`. |
| `client.window.width` / `height` | Fixed main popup size. Defaults to `360 × 240`. |
| `client.window.title` | Parsed metadata. The popup is borderless, so draw the visible title in HTML. |
| `client.window.opacity` | Whole-popup opacity from `0` (invisible) to `255` (opaque). Defaults to `235`. |
| `client.launcher` | Presence enables the optional launcher. See [Launcher](#optional-launcher). |
| `client.launcher.opacity` | Whole-launcher opacity from `0` to `255`. Defaults to `245`. |
| `entryPoints.hotkey` | Main-window toggle key: `F1`–`F12`, one letter, or one digit. |
| `entryPoints.uiButton.label` | Parsed label metadata. |
| `permissions` | Requested server capabilities. Effective set = manifest request ∩ operator policy. |

### Reserved or metadata-only fields

These fields may appear in manifests but are not current runtime controls:

| Field | Status |
|---|---|
| `manifest`, `author`, `description` | Metadata only. |
| `client.entry` | Reserved; client Lua is not executed. Omit it. |
| `client.html` | Parsed but the host currently opens fixed `client/index.html`. |
| `client.assets` | Not required; every safe file under `client/` is packed automatically. |
| `client.window.html`, `resizable` | Not active. The main window is fixed-size and uses `client/index.html`. |
| `client.launcher.html` | Not active as a path override. Use `client/launcher.html`. |
| `entryPoints.uiButton.icon`, `tooltip` | Metadata only in the current client host. |
| `limits.maxAssetBytes`, `limits.maxMsgBytes`, `limits.maxPending` | Not read from the manifest. Fixed host limits are listed in [Fixed limits](#fixed-limits). |

Do not depend on a reserved field merely because it parses as valid JSON.

## Capabilities and `policy.json`

Capabilities are denied by default. A capability is active only when it appears in both places:

1. the plugin's `permissions` array;
2. the operator's `Data/ClientLuaPlugin/policy.json` entry for the same plugin id.

```json
{
  "com.example.ranking": ["db.query", "player.readBasic"]
}
```

The currently enforced server capabilities are:

| Capability | Enables |
|---|---|
| `db.query` | `ctx:sql(query, params, callback)` |
| `player.readBasic` | `ctx:playerName()` and `ctx:playerLevel()` |
| `net.serverPush` | Provisional `ctx:push(...)` sender. No JavaScript push callback exists yet. |
| `ui.window` | Provisional `ctx:open()` and `host.open` senders. Server-initiated open is not executed by the current client host. |

`MUPF.invoke`, the manifest hotkey, and client-side `MUPF.open()` are current client-host features; `net.request` and `ui.hotkey` are not presently enforced capability gates. They may remain as descriptive metadata, but do not treat them as security boundaries.

Calling a capability-gated `ctx:*` method without the capability raises a Lua error and logs the failure.

## Optional launcher

Adding a `client.launcher` object enables a small launcher window. The page must be named `client/launcher.html`.

```json
{
  "client": {
    "window": { "width": 420, "height": 300, "title": "My Plugin", "opacity": 235 },
    "launcher": {
      "width": 150,
      "height": 46,
      "anchor": "top-left",
      "x": 16,
      "y": 96,
      "opacity": 245
    }
  }
}
```

Supported anchors:

`top-left`, `top-right`, `top-center`, `bottom-left`, `bottom-right`, `bottom-center`, `left`, `right`, `center`.

The launcher is fixed and non-closable. Its JavaScript opens the plugin's main window:

```js
function __mupf_click(x, y) {
  MUPF.open();
}
```

Both launcher and main popup follow the game window and are hidden outside the in-game world, while the game is minimized or tray-hidden, and while the Cash Shop is open.

`opacity` controls the alpha of the entire native plugin window, including its HTML content. It must be an integer. Values below `0` or above `255` are clamped, while a missing or non-integer value uses the legacy default (`235` for the main popup and `245` for the launcher). Use CSS colors or alpha inside the page when only an individual element should be transparent.

## Development and reload behavior

With `MUPF_DEV_HOTRELOAD = 1`, the GameServer rescans and repacks plugin folders when a player connects. The client development loop is:

```text
edit client/* -> reconnect/relog -> see the new UI
```

Changing `server.lua`, `plugin.json`, `policy.json`, or the installed plugin list requires the GameServer's **Reload Script** command or a GameServer restart. **Reload Script** rebuilds the GameServer Lua state and reloads plugin server environments without restarting the process.

The current reload path does not push replacement packages to players who are already online. Those players must reconnect to receive updated client files.

Window and launcher opacity are sent with the plugin BEGIN metadata. After changing an opacity value, reload scripts (or restart the GameServer) and reconnect the client so the native windows are recreated with the new value.

`MUPF_DEV_HOTRELOAD` is a build-time setting. It is not a manifest field or a per-plugin switch.

## Packaging and deployment

### Source-folder deployment

Source folders are the complete supported form for plugins that contain `server.lua`:

```text
Data/ClientLuaPlugin/com.example.plugin/plugin.json
Data/ClientLuaPlugin/com.example.plugin/server/server.lua
Data/ClientLuaPlugin/com.example.plugin/client/index.html
```

This form keeps `server.lua` on the GameServer and streams only `plugin.json` + `client/*`.

### Current `.mupf` behavior

The current command-line `MupfPacker` calls the shared client packer. It produces:

```text
MUPF header + obfuscated ZIP { plugin.json, client/* }
```

It does **not** include `server/*`. Therefore:

- use `.mupf` for client-only plugins whose manifest omits `server.entry`;
- keep server-backed plugins as source-folder deployments in this release;
- do not delete a server-backed source folder after creating a package with the current tool;
- do not install a folder and `.mupf` with the same plugin id as two separate entries.

The loader can read a full archive containing `server/*`, but the current public command-line packer does not generate that archive. Documentation will promote full single-file server-backed deployment only after the packer and loader contracts are aligned.

### Security note

The `MUPF` wrapper uses an embedded-key XOR keystream over the ZIP payload. This is package obfuscation and tamper friction, not a guarantee of secrecy. The decoding code necessarily exists in distributed binaries. Keep credentials, SQL policy, validation rules, and valuable logic in `server.lua`, which is not streamed to players.

Plain ZIP input is accepted by the loader for development compatibility. Do not use that compatibility behavior as a production security assumption.

## Fixed limits

The current host enforces fixed limits; manifest `limits.*` does not override them.

| Limit | Value | Effect |
|---|---:|---|
| Plugins loaded by one GameServer | 64 | Additional entries are not loaded. |
| Uncompressed files per plugin | 256 | Package is rejected above the cap. |
| Total uncompressed plugin bytes | 4 MiB | Package is rejected/quarantined. |
| One uncompressed entry | 4 MiB | Package is rejected. |
| VFS path length | 255 bytes | Entry is rejected. Traversal, absolute, drive, and backslash paths are also rejected. |
| Request/reply payload | 8,000 bytes | Envelope encode/decode fails above the cap. |
| Function/channel name | 64 bytes | Envelope is rejected above the cap. |
| Pending requests per plugin instance | 64 | `MUPF.invoke` returns `0` when busy. |
| Request timeout | 10 seconds | The pending request expires. |
| Calls per player and plugin | 20 per 1-second window | Excess request receives a rate-limit error when a reply was expected. |
| JSON nesting | 32 levels | Deeper values are converted to `null`. |

Paginate server responses and keep UI assets small even when they are below the hard ceilings.

## Pre-release checklist

- [ ] `plugin.json.id` exactly matches `PluginRegister(...)`.
- [ ] Main UI is `client/index.html`; optional launcher is `client/launcher.html`.
- [ ] No client-side Lua is expected.
- [ ] Every request function and argument is validated in `server.lua`.
- [ ] SQL uses `ctx:sql` placeholders, never client-built SQL text.
- [ ] Manifest and policy grant only the server capabilities the plugin uses.
- [ ] Reply payloads remain under 8,000 bytes.
- [ ] Client assets remain under the fixed package limits.
- [ ] Client changes were tested after a fresh reconnect.
- [ ] Server changes were tested after **Reload Script** or restart.
- [ ] A server-backed plugin is deployed as a source folder with the current packer release.

## See also

- [MUPF System Overview](MUPF-Client-System-Overview.md)
- [Writing Plugins](Writing-Plugins.md)
- [MUPF Server API](MUPF-Server-API.md)
- [MUPF Client API](MUPF-Client-API.md)
