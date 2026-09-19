# Writing MUPF Plugins

This guide builds a working external MUPF plugin from an empty folder. It intentionally keeps the tutorial separate from the authoritative references:

- [MUPF Plugins](MUPF-Plugins.md) — manifest, capabilities, limits, reload, and deployment;
- [MUPF Server API](MUPF-Server-API.md) — `PluginRegister`, `host`, and `ctx:*`;
- [MUPF Client API](MUPF-Client-API.md) — JavaScript UI runtime.

> The current client UI runtime is HTML/CSS/JavaScript. There is no client-side Lua. The old `Data/LuaUI` system is a separate compatibility path and is not used by this tutorial.

## 1. Create the plugin folder

Create this structure under the server data directory:

```text
Data/ClientLuaPlugin/
└─ com.example.hello/
   ├─ plugin.json
   ├─ server/
   │  └─ server.lua
   └─ client/
      └─ index.html
```

The entry paths are fixed in the current host:

- main UI: `client/index.html`;
- optional launcher: `client/launcher.html`.

## 2. Add `plugin.json`

```json
{
  "id": "com.example.hello",
  "name": "Hello World",
  "version": "1.0.0",
  "apiVersion": 1,
  "minApiVersion": 1,

  "server": {
    "entry": "server/server.lua"
  },

  "client": {
    "window": {
      "width": 360,
      "height": 200,
      "title": "Hello World",
      "opacity": 235
    }
  },

  "entryPoints": {
    "hotkey": "F7"
  },

  "permissions": []
}
```

Request/reply itself does not currently require a capability. Add permissions only for capability-gated server work such as `db.query` or `player.readBasic`.

The visible popup is borderless, so draw your title in HTML. The manifest `title` is metadata; `width` and `height` control the fixed popup size. `opacity` controls the whole native popup from `0` (invisible) to `255` (opaque) and defaults to `235` when omitted.

## 3. Add `server/server.lua`

```lua
local Hello = {}

function Hello.OnLoad()
    host.log("hello plugin loaded")
end

function Hello.OnInvoke(ctx, fn, args, reqId)
    -- fn and args came from the client. Treat both as untrusted.
    if fn ~= "hello" then
        ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
        return
    end

    local name = "stranger"
    if type(args) == "table" and type(args.name) == "string" then
        name = string.sub(args.name, 1, 24)
    end

    ctx:reply(reqId, { greeting = "Hello, " .. name .. "!" })
end

PluginRegister("com.example.hello", Hello)
return Hello
```

The id passed to `PluginRegister` must exactly match `plugin.json.id`.

## 4. Add `client/index.html`

MUPF uses litehtml for layout and duktape for JavaScript. It is not a browser:

- there is no mutable DOM;
- there is no `document.getElementById` or `addEventListener`;
- there are no timers, `fetch`, XHR, WebSocket, local storage, or cookies;
- clicks arrive through `__mupf_click(x, y)`;
- the page updates by rebuilding HTML and calling `MUPF.render(html)`.

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"></head>
<body>
  <div>Loading...</div>
  <script>
    var W = 360, H = 200;
    var message = 'Click GREET to call the GameServer.';

    var CSS =
      'html,body{margin:0;padding:0;font:13px "Segoe UI";color:#e8edf4}' +
      'body{background:#11151c;border:1px solid #40566e}' +
      '#title{height:36px;display:flex;align-items:center;justify-content:center;' +
      'background:linear-gradient(#315c83,#1c334b);font-weight:700}' +
      '#body{padding:24px 16px;text-align:center}' +
      '#button{position:absolute;left:20px;right:20px;bottom:18px;height:40px;' +
      'display:flex;align-items:center;justify-content:center;background:#d8a63c;' +
      'border:1px solid #805a18;color:#201600;font-weight:700}';

    function esc(value) {
      return String(value == null ? '' : value)
        .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
    }

    function view() {
      return '<style>' + CSS + '</style>' +
        '<div id="title">HELLO WORLD</div>' +
        '<div id="body">' + esc(message) + '</div>' +
        '<div id="button">GREET</div>';
    }

    function render() {
      MUPF.render(view());
    }

    function __mupf_click(x, y) {
      if (y >= H - 58) {
        MUPF.invoke('hello', { name: 'Player' }, function (response) {
          message = response && response.greeting
            ? response.greeting
            : 'Error: ' + String(response && response.error || 'unknown');
          render();
        });
      }
    }

    render();
  </script>
</body>
</html>
```

Keep the executable script in the entry page. Strings passed to `MUPF.render()` should contain markup and styles, not another script block. JavaScript globals remain alive between renders.

## 5. Run the plugin

1. Start or reload the GameServer after adding the new plugin.
2. Confirm the plugin log contains `hello plugin loaded`.
3. Connect or reconnect the client so it receives the client package.
4. Enter the world and press `F7`.
5. Click **GREET** and verify the request/reply round trip.

During development, client changes follow this loop:

```text
edit client/* -> reconnect/relog -> retest
```

Server Lua, manifest, policy, and plugin-list changes require **Reload Script** or a GameServer restart. Client assets are not repushed to players who are already online.

## 6. Add capability-gated player data

To use `ctx:playerName()` or `ctx:playerLevel()`, request the capability:

```json
{
  "permissions": ["player.readBasic"]
}
```

Then grant it in the operator policy:

```json
{
  "com.example.hello": ["player.readBasic"]
}
```

Now server Lua can read the caller:

```lua
local name = ctx:playerName() or "unknown"
local level = ctx:playerLevel()
ctx:reply(reqId, { name = name, level = level })
```

The effective capability set is always:

```text
plugin.json permissions ∩ policy.json grants
```

No policy entry means no capability-gated server methods.

## 7. Query the database safely

Request and grant `db.query`, then use placeholders:

```lua
function Plugin.OnInvoke(ctx, fn, args, reqId)
    if fn ~= "getRanks" then
        ctx:reply(reqId, { error = "unknown_fn" })
        return
    end

    ctx:sql(
        "SELECT name, level FROM character_info WHERE authority = ? ORDER BY level DESC LIMIT 100",
        { 0 },
        function(rows)
            local result = {}
            for i, row in ipairs(rows) do
                result[i] = {
                    rank = i,
                    name = row.name or "",
                    level = tonumber(row.level) or 0
                }
            end
            ctx:reply(reqId, { rows = result })
        end)
end
```

This read-only example can show an empty list when the SELECT failed: current
`ctx:sql` sends an empty table both for a failed SELECT and for zero rows.
It does not send `nil` on failure. Do not use a `nil` check or this callback
alone to decide whether to consume items, charge currency, or issue a refund.
See [MUPF Server API](MUPF-Server-API.md#ctxsqlquery-params-callback).

Rules:

- never concatenate client input into SQL;
- validate and clamp every client argument before using it;
- keep literal `?` characters out of SQL string literals/comments because placeholders are replaced lexically;
- paginate large results so each JSON reply stays below 8,000 bytes.

## 8. Add an optional launcher

Add launcher geometry to the manifest:

```json
{
  "client": {
    "window": { "width": 420, "height": 300, "opacity": 235 },
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

Launcher opacity also uses `0..255` and defaults to `245`. Integer values outside the range are clamped. To make only the launcher artwork translucent while keeping the native window fully opaque, leave the manifest value at `255` and use CSS alpha colors in `launcher.html`.

Create `client/launcher.html`:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <style>
    html,body{margin:0;padding:0}
    #open{width:146px;height:42px;display:flex;align-items:center;justify-content:center;
      background:linear-gradient(#2d3645,#151a22);border:2px solid #d2a33f;
      color:#f0e4ca;font:700 13px "Segoe UI"}
  </style>
</head>
<body>
  <div id="open">OPEN</div>
  <script>
    function __mupf_click(x, y) { MUPF.open(); }
  </script>
</body>
</html>
```

The launcher is fixed and non-closable. `MUPF.open()` without an id opens this plugin's main popup. `MUPF.open("com.example.other")` opens another locally loaded plugin's main popup.

Server-initiated open (`ctx:open` / `host.open`) is provisional and does not currently open the client popup.

## 9. Handle text input

There is no browser form-control event model. Capture supported keys explicitly:

```js
var editing = false;
var value = '';

function focusField() {
  editing = true;
  MUPF.captureKeys(true);
}

function __mupf_key(vk) {
  if (!editing) return;

  if (vk === 13) {
    editing = false;
    MUPF.captureKeys(false);
    submitValue();
  } else if (vk === 8) {
    value = value.slice(0, -1);
  } else if (vk >= 48 && vk <= 57 && value.length < 10) {
    value += String.fromCharCode(vk);
  }

  render();
}
```

The host forwards letters, digits, Backspace, Enter, Space, Delete, minus, and decimal keys. Escape, arrow keys, and function keys continue to reach the game. Capture is released when the popup closes.

## 10. UI rules that prevent most bugs

- Build the whole view as a string and call `MUPF.render`.
- Keep application state in JavaScript globals.
- Escape every server/client string before inserting it into HTML.
- Use `__mupf_click(x, y)` for hit testing.
- Reserve roughly the top 38 px for dragging and the top-right 38 × 38 px for close.
- Use fixed pixel layouts matching the manifest window size.
- Do not rely on DOM APIs, timers, network APIs, hover, animations, or live form controls.
- Re-include external CSS links or inline styles in every rendered view.
- Use SVG for crisp icons; PNG, JPG, and BMP are also supported. Raster images are limited to 2048 × 2048.
- SVG `<text>` is not rendered; place text in HTML.
- Avoid unsupported CSS such as `border-radius`, `box-shadow`, radial gradients, transitions, and `:hover`.

## 11. Fixed host limits

Manifest `limits.*` fields do not currently override these values:

| Resource | Limit |
|---|---:|
| Request/reply payload | 8,000 bytes |
| Function/channel name | 64 bytes |
| Pending requests per plugin instance | 64 |
| Request timeout | 10 seconds |
| Requests per player/plugin | 20 per second |
| Total uncompressed client package | 4 MiB |
| One package entry | 4 MiB |
| Package entries | 256 |
| VFS path | 255 bytes |
| JSON nesting | 32 levels |

## 12. Packaging and production

The current command-line `MupfPacker` packages `plugin.json` and `client/*`. It does not include `server/*`.

Therefore:

- deploy server-backed plugins as source folders in the current release;
- use `.mupf` only for client-only plugins whose manifest omits `server.entry`;
- never install both a folder and package as duplicate entries with the same plugin id;
- treat package obfuscation as a deterrent, not secret storage.

Set `MUPF_DEV_HOTRELOAD = 0` in a production GameServer build to avoid repacking plugin folders on every connection. This is a build-time setting controlled by the server distributor, not a plugin manifest option.

## 13. Troubleshooting

| Symptom | Check |
|---|---|
| Plugin does not appear in server logs | Folder is under `Data/ClientLuaPlugin/` and contains valid `plugin.json`. |
| Client window never opens | `client/index.html` exists, hotkey is valid, player reconnected, and client API requirement is not newer than the host. |
| Launcher is absent | `client.launcher` exists in the manifest and `client/launcher.html` exists. |
| Request returns handler error | `PluginRegister` ran, id matches the manifest, and `OnInvoke` exists. |
| `ctx:*` throws a capability error | Permission is present in both manifest and `policy.json`. |
| Client edit is not visible | Reconnect/relog; online clients keep the old package. |
| Server edit is not visible | Use **Reload Script** or restart the GameServer. |
| UI stays on Loading | Check JavaScript syntax and avoid stray literal script tags in comments/text that confuse the simple script extractor. |
| Reply never arrives | Keep payload below 8,000 bytes, call `ctx:reply` once, and check rate-limit/server logs. |
| Server push/open does nothing | Those server-originated client actions are provisional. Use request/reply and client-side open. |

## 14. Pre-release checklist

- [ ] Manifest id exactly matches `PluginRegister`.
- [ ] Main page is `client/index.html`; launcher is `client/launcher.html`.
- [ ] No client-side Lua is expected.
- [ ] Client UI uses only the documented MUPF JavaScript bridge.
- [ ] Every server request validates `fn`, types, ranges, ownership, and resources.
- [ ] SQL uses `ctx:sql` placeholders.
- [ ] Capabilities are minimal and granted in both manifest and policy.
- [ ] Replies are paginated and remain below 8,000 bytes.
- [ ] Assets and package size remain below fixed host caps.
- [ ] Client changes were tested after reconnect.
- [ ] Server changes were tested after **Reload Script** or restart.
- [ ] Server-backed plugin is deployed as a source folder with the current packer release.
