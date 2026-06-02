# Writing Plugins

A complete, practical guide to building **MU Plugin Framework (MUPF)** plugins — from a first hello-world to a full multi-window, database-backed plugin of any complexity. For the API reference, see [MUPF Plugins](MUPF-Plugins.md), [MUPF Server API](MUPF-Server-API.md), and [MUPF Client API](MUPF-Client-API.md).

> **Architecture in one line:** the **front-end is HTML/JavaScript** (your `client/` pages), the
> **logic and data live on the server** (`server.lua`), and the client DLL's **Lua bridge**
> connects them (`MUPF.invoke` ↔ `OnInvoke` / `ctx:reply`). There is **no client-side Lua
> runtime** — you don't write client Lua; `client.entry`/`client.lua` is a reserved manifest field.

## Contents

- [1. Quickstart — your first plugin](#1-quickstart--your-first-plugin)
- [2. Building the UI (client side)](#2-building-the-ui-client-side)
- [3. The server side (server.lua)](#3-the-server-side-serverlua)
- [4. The 2-tier launcher (always-on button)](#4-the-2-tier-launcher-always-on-button)
- [5. Packaging, shipping & operations](#5-packaging-shipping--operations)
- [6. Recipes — patterns for real plugins](#6-recipes--patterns-for-real-plugins)
- [7. Limits, gotchas & a pre-ship checklist](#7-limits-gotchas--a-pre-ship-checklist)

---

## 1. Quickstart — your first plugin

This walks you from an empty folder to a working plugin that does a real **client → server → client** round-trip: a button calls the server, the server replies with a greeting, and the page repaints with it. No packing, no build — drop the folder in, relog, done.

If you only read one thing first, read the **[no-live-DOM model](MUPF-Client-API.md#-important-model--there-is-no-live-dom)** — it is the single biggest difference from web dev. Everything below assumes it.

### What you'll build

A plugin `com.example.hello` with one server function, `hello`, that returns a greeting. The client window shows a button; clicking it calls `MUPF.invoke("hello", …)` and renders the reply.

```
[click button] → MUPF.invoke("hello", {}) → (server) OnInvoke → ctx:reply
              → (client) cb(resp) → MUPF.render(view())   // page repaints
```

### Step 1 — Folder layout

A plugin is **one folder** with three parts. Create it under the server's plugins folder, `Data/ClientLuaPlugin/`:

```
Data/ClientLuaPlugin/
└─ com.example.hello/
   ├─ plugin.json          ← manifest: id, window, hotkey, permissions
   ├─ server/
   │  └─ server.lua        ← runs on the GameServer (never sent to the client)
   └─ client/
      └─ index.html        ← the in-game UI (HTML/CSS/JS), the entry page
```

| Part | Runs where | Notes |
|---|---|---|
| `plugin.json` | parsed by both | The `id` is the routing + policy key. **Must** match the id you pass to `PluginRegister`. |
| `server/server.lua` | GameServer Lua state | Never streamed to players. Loaded once at startup. |
| `client/index.html` | in-game window (litehtml + duktape) | Streamed to the player on connect. This is the **entry** page and the only one that may carry `<script>`. |

💡 The folder name does not have to equal the `id`, but matching them keeps things obvious. The `id` inside `plugin.json` is what actually matters.

### Step 2 — The manifest (`plugin.json`)

Minimal, but every field here is load-bearing. See the full field table in **[MUPF Plugins → The manifest](MUPF-Plugins.md#the-manifest--pluginjson)**.

`Data/ClientLuaPlugin/com.example.hello/plugin.json`:

```json
{
  "manifest": 1,
  "id": "com.example.hello",
  "name": "Hello World",
  "version": "1.0.0",
  "apiVersion": 1,
  "minApiVersion": 1,
  "author": "You",
  "description": "Minimal hello-world plugin: a button that greets you from the server.",

  "server": { "entry": "server/server.lua" },

  "client": {
    "html": "client/index.html",
    "window": { "width": 360, "height": 200, "resizable": false, "title": "Hello World" }
  },

  "entryPoints": {
    "hotkey": "F7",
    "uiButton": { "icon": null, "label": "Hello", "tooltip": "Say hello" }
  },

  "permissions": ["ui.window", "ui.hotkey", "net.request"]
}
```

What the minimal set buys you:

- `server.entry` — where the server Lua lives.
- `client.html` — the entry page. `client.window` gives the window its **fixed** size and title.
- `entryPoints.hotkey` — the key that toggles the window in-game (here `F7`).
- `permissions` — what the plugin *requests*. For a client→server round-trip you need at minimum `net.request` (the client may call the server) and `ui.window` (the window may open). `ui.hotkey` enables the hotkey. We don't need `db.query` here because hello-world doesn't touch the database.

⚠️ Permissions in the manifest are only a **request**. A capability is granted only if it's also in the operator's `policy.json` (Step 4). Manifest ∩ policy, **deny-by-default** — an ungranted `ctx:*` call raises a Lua error that's caught and logged, and the call fails.

### Step 3 — The server (`server/server.lua`)

The server side registers a hooks table with `PluginRegister(id, hooks)` and handles requests in `OnInvoke(ctx, fn, args, reqId)`. The skeleton and the full `ctx` reference are in **[MUPF Server API](MUPF-Server-API.md)**.

`Data/ClientLuaPlugin/com.example.hello/server/server.lua`:

```lua
-- Hello World — minimal server side. One function, "hello", returns a greeting.
local Hello = {}

function Hello.OnLoad()
    host.log("hello server ready")            -- optional; runs once at load, writes to the 'plugin' log channel
end

-- A client MUPF.invoke(...) arrived. fn/args are UNTRUSTED — validate before use.
function Hello.OnInvoke(ctx, fn, args, reqId)
    if fn == "hello" then
        -- args is the client's payload, already parsed into a Lua table.
        local who = (args and type(args.name) == "string" and args.name ~= "") and args.name or "stranger"
        ctx:reply(reqId, { greeting = "Hello, " .. who .. "!" })   -- table -> JSON -> client cb
        return
    end
    -- Unknown function: reply with your own error shape (the client checks resp.error).
    ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
end

PluginRegister("com.example.hello", Hello)    -- id MUST equal plugin.json "id"
return Hello
```

Things that bite first-timers:

- **`PluginRegister` id must equal the manifest `id`.** The host only lets the plugin whose `server.lua` is *currently loading* register under its own id; a mismatch is rejected and logged, and the plugin gets no invocations.
- **A plugin with no `PluginRegister` / no `OnInvoke` returns `no_handler`** to every client call (the client's `cb` gets `{ error: "no_handler" }`).
- **Call `ctx:reply(reqId, table)` exactly once** per `reqId`. The body must be a table (or omitted); it's serialized to JSON and delivered to the client callback. An empty Lua table `{}` serializes as `[]`, so use a keyed table when you want a JSON object.
- **Treat `fn` and `args` as hostile.** A modified client can send any `fn`/`args`. Validate types and ranges (the snippet above checks `args.name` is a non-empty string) — see [JSON ↔ Lua](MUPF-Server-API.md#json--lua).
- `host.log(...)` needs no capability and writes to the server's `plugin` log channel — your first debugging tool.

### Step 4 — Grant the capabilities (`policy.json`)

The operator's policy file decides what each plugin id is *actually* allowed to do. It lives **once** at the root of the plugins folder, keyed by plugin id:

`Data/ClientLuaPlugin/policy.json`:

```json
{
  "com.example.hello": ["ui.window", "ui.hotkey", "net.request"]
}
```

🔒 No `policy.json`, or a plugin id absent from it, means **all capabilities denied** for that plugin. The granted set is the **intersection** of the manifest `permissions` and the policy array — so list here exactly the caps your manifest requests and you want live. If you later add `db.query` to the manifest, you must also add it here or `ctx:sql` will error.

If you already have a `policy.json` (e.g. for the bundled Ranking plugin), just add your plugin's line as another key — it's one object mapping every plugin id to its allowed caps.

### Step 5 — The client (`client/index.html`)

The client window is drawn by litehtml with JS run by duktape — **not a browser**. There's no live DOM, no `getElementById`, no `addEventListener`, no `setTimeout`/`fetch`. You build a full HTML string and call `MUPF.render(html)` to repaint; state lives in **JS globals** that survive across renders. Clicks arrive as coordinates via `__mupf_click(x, y)` — you hit-test regions yourself. Read **[MUPF Client API](MUPF-Client-API.md)** for the full model.

Three host-provided JS calls are all you need here:

| Call | Does |
|---|---|
| `MUPF.invoke(fn, args, cb)` | Calls the server's `OnInvoke`; `cb(resp)` fires with the parsed reply (or `{ error: … }`). Needs `net.request`. |
| `MUPF.render(html)` | Replaces the whole page with `html`. Applied next frame, safe to call from a callback. The rendered string must contain **no `<script>`** — only this entry page does. |
| `MUPF.close()` | Closes the window. |

`Data/ClientLuaPlugin/com.example.hello/client/index.html`:

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"></head>
<body>
  <!-- No live DOM: build HTML strings + MUPF.render(); state lives in JS globals.
       The entry page is the only page that carries <script>. -->
  <div id="boot">Hello World loading&#8230;</div>
  <script>
    // Window size from plugin.json client.window. Used for click hit-testing.
    var W = 360, H = 200;

    // State persists across MUPF.render() (same JS context between repaints).
    var greeting = 'Click GREET to call the server.';

    var CSS =
      'html,body{margin:0;padding:0;font:13px "Segoe UI",Tahoma,sans-serif;color:#d7dbe4;-webkit-user-select:none;user-select:none}' +
      'body{background:#0e0f13;border:1px solid #2b2f3a}' +
      '#wrap{position:relative;height:194px;background:linear-gradient(#1b1c22,#121319)}' +
      '#tb{position:relative;height:36px;display:flex;align-items:center;justify-content:center;background:linear-gradient(#2f5e8e,#1c3656);border-bottom:2px solid #4a7fb3}' +
      '#tt{color:#e7eef6;font-size:14px;font-weight:700;letter-spacing:1.5px;text-shadow:0 1px 2px #000}' +
      '#x{position:absolute;top:0;right:0;width:36px;height:36px;display:flex;align-items:center;justify-content:center;color:#a6c2f3;font-size:16px;font-weight:bold}' +
      '#bd{padding:22px 18px;text-align:center}' +
      '.msg{color:#cfd3dd;font-size:14px;line-height:1.5;min-height:40px}' +
      '#btn{position:absolute;left:16px;right:16px;bottom:16px;height:40px;display:flex;align-items:center;justify-content:center;' +
      'background:linear-gradient(#e8b54a,#c8922e);color:#2a1d05;font-size:14px;font-weight:700;letter-spacing:2px;border:1px solid #8a5a14}';

    // Always escape any text that came from the server before putting it in markup.
    function esc(s){ return String(s==null?'':s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }

    // Build the page as a STRING (no DOM). MUPF.render re-parses + repaints.
    function view(){
      return '<!DOCTYPE html><html><head><meta charset="utf-8"><style>'+CSS+'</style></head><body><div id="wrap">'+
        '<div id="tb"><span id="tt">HELLO WORLD</span><div id="x">&#10005;</div></div>'+   // U+2715 ✕ renders
        '<div id="bd"><div class="msg">'+esc(greeting)+'</div></div>'+
        '<div id="btn">GREET</div>'+
        '</div></body></html>';
    }
    function render(){ MUPF.render(view()); }

    // No DOM => clicks come as (x,y) in client px; 0,0 = window top-left. Hit-test regions.
    // The host reserves the top ~38px as a drag strip and the top-right ~38x38 as a close hotspot.
    function __mupf_click(x, y){
      if (y < 36 && x >= W-38) { MUPF.close(); return; }          // top-right = close
      if (y >= H-56) {                                            // bottom button = greet
        MUPF.invoke('hello', {}, function(resp){                  // {} -> no name -> "stranger"
          if (!resp || resp.error) { greeting = 'Error: ' + esc(resp && resp.error || 'unknown'); render(); return; }
          greeting = resp.greeting;                               // server reply
          render();                                               // repaint with it
        });
      }
    }

    render();   // first paint — no server call needed to show the UI
  </script>
</body>
</html>
```

How a click round-trips: `__mupf_click(x,y)` → `MUPF.invoke('hello', …)` → server `OnInvoke` → `ctx:reply(reqId, { greeting })` → your `cb(resp)` → `render()` repaints with `resp.greeting`. (To pass a name, send `{ name: 'YourName' }` instead of `{}`.)

⚠️ The string you pass to `MUPF.render()` must contain **no `<script>`** — keep all JS in this entry `index.html`. Re-rendered views are pure markup; your globals and functions stay alive between renders.

### Step 6 — Run it in DEV mode (no packing)

In dev you work straight from the source folder — **no `.mupf` packing, no full restart**:

1. **Drop the folder** `com.example.hello/` into `Data/ClientLuaPlugin/` (next to `policy.json`).
2. **Make sure the server is in dev hot-reload mode** — `MUPF_DEV_HOTRELOAD = 1` in `Game/PluginMgr.cpp` (the default in dev builds). In dev, the server re-reads each plugin from disk on connect.
3. **Relog.** On reconnect the server streams the fresh `client/` files; press the hotkey (`F7`) and the window appears. Click **GREET** to see the server round-trip.

The dev loop for the **client** side is just:

```
edit client/index.html  →  relog  →  see the change
```

⚠️ The **server** side is different. `server.lua` is loaded **once into the Lua stack at startup**, so after editing `server.lua` (or adding/removing a plugin) you need a server restart — or use the GameServer window's **`Reload Script`** command, which re-scans and reloads every plugin's `server.lua` without restarting the GameServer (online players keep the old UI until they relog; new logins get the fresh client). See [MUPF Plugins → Developing a plugin](MUPF-Plugins.md#developing-a-plugin-dev-vs-ship).

💡 First thing to check if nothing shows: the server `plugin` log channel. `OnLoad`'s `host.log("hello server ready")` confirms `server.lua` loaded and registered; a `cap '…' NOT granted by policy.json` line means you forgot Step 4; a `no_handler` reply to the client means `PluginRegister` didn't run (usually an id mismatch or a compile error in `server.lua`).

When the plugin is ready to ship, you pack the whole folder into one encrypted `.mupf` with **MupfPacker** and set `MUPF_DEV_HOTRELOAD = 0` — see [MUPF Plugins → Ship](MUPF-Plugins.md#ship--pack-into-one-mupf). The same code path loads both folders (dev) and `.mupf` files (shipped) from the same directory, so you can mix them.

### The complete hello-world

Copy these four files verbatim and you have a working plugin.

`Data/ClientLuaPlugin/com.example.hello/plugin.json`:

```json
{
  "manifest": 1,
  "id": "com.example.hello",
  "name": "Hello World",
  "version": "1.0.0",
  "apiVersion": 1,
  "minApiVersion": 1,
  "author": "You",
  "description": "Minimal hello-world plugin: a button that greets you from the server.",
  "server": { "entry": "server/server.lua" },
  "client": {
    "html": "client/index.html",
    "window": { "width": 360, "height": 200, "resizable": false, "title": "Hello World" }
  },
  "entryPoints": {
    "hotkey": "F7",
    "uiButton": { "icon": null, "label": "Hello", "tooltip": "Say hello" }
  },
  "permissions": ["ui.window", "ui.hotkey", "net.request"]
}
```

`Data/ClientLuaPlugin/com.example.hello/server/server.lua`:

```lua
local Hello = {}

function Hello.OnLoad()
    host.log("hello server ready")
end

function Hello.OnInvoke(ctx, fn, args, reqId)
    if fn == "hello" then
        local who = (args and type(args.name) == "string" and args.name ~= "") and args.name or "stranger"
        ctx:reply(reqId, { greeting = "Hello, " .. who .. "!" })
        return
    end
    ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
end

PluginRegister("com.example.hello", Hello)
return Hello
```

`Data/ClientLuaPlugin/com.example.hello/client/index.html`:

```html
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"></head>
<body>
  <div id="boot">Hello World loading&#8230;</div>
  <script>
    var W = 360, H = 200;
    var greeting = 'Click GREET to call the server.';

    var CSS =
      'html,body{margin:0;padding:0;font:13px "Segoe UI",Tahoma,sans-serif;color:#d7dbe4;-webkit-user-select:none;user-select:none}' +
      'body{background:#0e0f13;border:1px solid #2b2f3a}' +
      '#wrap{position:relative;height:194px;background:linear-gradient(#1b1c22,#121319)}' +
      '#tb{position:relative;height:36px;display:flex;align-items:center;justify-content:center;background:linear-gradient(#2f5e8e,#1c3656);border-bottom:2px solid #4a7fb3}' +
      '#tt{color:#e7eef6;font-size:14px;font-weight:700;letter-spacing:1.5px;text-shadow:0 1px 2px #000}' +
      '#x{position:absolute;top:0;right:0;width:36px;height:36px;display:flex;align-items:center;justify-content:center;color:#a6c2f3;font-size:16px;font-weight:bold}' +
      '#bd{padding:22px 18px;text-align:center}' +
      '.msg{color:#cfd3dd;font-size:14px;line-height:1.5;min-height:40px}' +
      '#btn{position:absolute;left:16px;right:16px;bottom:16px;height:40px;display:flex;align-items:center;justify-content:center;' +
      'background:linear-gradient(#e8b54a,#c8922e);color:#2a1d05;font-size:14px;font-weight:700;letter-spacing:2px;border:1px solid #8a5a14}';

    function esc(s){ return String(s==null?'':s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }

    function view(){
      return '<!DOCTYPE html><html><head><meta charset="utf-8"><style>'+CSS+'</style></head><body><div id="wrap">'+
        '<div id="tb"><span id="tt">HELLO WORLD</span><div id="x">&#10005;</div></div>'+
        '<div id="bd"><div class="msg">'+esc(greeting)+'</div></div>'+
        '<div id="btn">GREET</div>'+
        '</div></body></html>';
    }
    function render(){ MUPF.render(view()); }

    function __mupf_click(x, y){
      if (y < 36 && x >= W-38) { MUPF.close(); return; }
      if (y >= H-56) {
        MUPF.invoke('hello', {}, function(resp){
          if (!resp || resp.error) { greeting = 'Error: ' + esc(resp && resp.error || 'unknown'); render(); return; }
          greeting = resp.greeting;
          render();
        });
      }
    }

    render();
  </script>
</body>
</html>
```

`Data/ClientLuaPlugin/policy.json` (add this key if the file already exists):

```json
{
  "com.example.hello": ["ui.window", "ui.hotkey", "net.request"]
}
```

Drop the folder in, ensure `policy.json` grants the caps, relog, press `F7`, click **GREET** — you should see *"Hello, stranger!"* come back from the server. From here, add `db.query` + `ctx:sql` for data (the bundled **Server Ranking** plugin is the reference), or a tier-1 launcher button (the **2-Tier Test** plugin). See [MUPF Plugins](MUPF-Plugins.md), [MUPF Server API](MUPF-Server-API.md), and [MUPF Client API](MUPF-Client-API.md).

---

## 2. Building the UI (client side)

Your plugin's UI is an HTML page (`client/index.html`, the **entry** page) rendered **in-game** in its own borderless window. It is drawn by the client's bundled engine — **litehtml** for HTML/CSS and **duktape** for JavaScript — *not* a browser. The single biggest adjustment from web dev: **there is no live DOM.** Internalize the re-render model below before you write a line of UI; everything else follows from it.

For the exhaustive API surface see [MUPF Client API](MUPF-Client-API.md); for the manifest/window/launcher fields see [MUPF Plugins](MUPF-Plugins.md). This section is the practical how-to.

### 2.1 The mental model: re-render, don't mutate

litehtml parses HTML and paints it, but it does **not** hand JavaScript a mutable document. None of these exist:

| You might reach for… | Reality | Do this instead |
|---|---|---|
| `document.getElementById`, `el.innerHTML = …`, `el.style.…` | no live DOM | rebuild a full HTML string + `MUPF.render(html)` |
| `addEventListener`, `el.onclick` | no DOM events | the global `__mupf_click(x, y)` + your own hit-testing |
| `setTimeout` / `setInterval` | not available | re-render in response to clicks / server replies only |
| `fetch` / `XMLHttpRequest` | not available | `MUPF.invoke(fn, args, cb)` to your own server Lua |

The page works by **re-rendering**: your JS builds the whole page as a string and calls `MUPF.render(htmlString)`. The host re-parses and repaints. Crucially, **your JS context survives between renders** — globals and functions defined in the entry page stay alive. So you keep all UI state in JS globals, and each `MUPF.render()` is a pure "given current state, draw the page" function.

```js
// ENTRY page only: define state + functions, then do the first paint/request.
var W = 420, H = 300;          // window size from plugin.json client.window
var rows = [], page = 0;       // <-- these PERSIST across every MUPF.render()

function esc(s){ return String(s==null?'':s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }

function view(){               // build the page as a STRING from current state
  var h = '<style>'+CSS+'</style><div id="bd">';
  for (var i=0;i<rows.length;i++) h += '<div>'+esc(rows[i].name)+'</div>';
  return h + '</div>';
}
function render(){ MUPF.render(view()); }   // re-parse + repaint
```

> ⚠️ **Re-rendered pages must carry NO `<script>`.** Only the *entry* `index.html` carries the `<script>` block. The host text-scans the entry page once and runs its script into the JS context; the strings you later pass to `MUPF.render()` are **pure markup**. Your functions and globals are already live — re-declaring them in a re-render string would do nothing useful and a `<script>` there is ignored at best. (As a corollary: never write a literal `<script` open tag inside an HTML *comment* in the entry page — the host scans by text, not by a real parser, and will mistake it for the start of a script block.)

> 💡 A clean pattern, used by both bundled plugins: a `view()`/`pageHtml()` builder that returns a **complete** `<!DOCTYPE html>…</html>` document (with the `<style>` inlined), and a thin `render()` wrapper that does `MUPF.render(view())`. State changes mutate globals, then call `render()`.

### 2.2 The `MUPF` object

The host injects a small `MUPF` shim. The four methods you use to build UI:

| Call | Effect | Capability |
|---|---|---|
| `MUPF.invoke(fn, args, cb)` → reqId | call your server `OnInvoke(ctx, fn, args, reqId)`; `cb(resp)` fires with the parsed reply (or `{error:…}`) | `net.request` |
| `MUPF.render(html)` | replace the whole page with `html` (applied next frame, off the JS stack — safe from a callback) | — |
| `MUPF.close()` | close this (main) plugin window | — |
| `MUPF.open([id])` | open this plugin's main window (`MUPF.open()`) or another's (`MUPF.open("com.x.y")`) — used by a tier-1 launcher | `ui.window` |

```js
MUPF.invoke('getRanks', { page: 0 }, function(resp){
  if (!resp || resp.error) { MUPF.render(errorView(resp && resp.error)); return; }
  rows = resp.rows || [];
  page = 0;
  render();
});
```

`MUPF.render()` is applied on the next draw frame, not synchronously — so calling it from inside an `invoke` callback (or `__mupf_click`) is safe and is the normal flow. The round-trip:

```
[click] → __mupf_click(x,y) → MUPF.invoke(fn,args,cb)
   → (server) OnInvoke(ctx,fn,args,reqId) → ctx:sql → ctx:reply(reqId,data)
   → (client) cb(data) → render()   // page repaints with the data
```

### 2.3 Receiving input — `__mupf_click(x, y)` and hit-testing

There are no DOM click handlers. The host forwards every content click to a **global** function `__mupf_click(x, y)` with **client pixel** coordinates (`0,0` = window top-left). You hit-test your own regions against the layout you drew:

```js
function __mupf_click(x, y){
  // top-right = close (only the host's close hotspot is reserved; you may also draw your own)
  if (y < 36 && x >= W-38) { MUPF.close(); return; }
  // bottom button strip = action
  if (y >= H-60) {
    MUPF.invoke('ping', { n: cnt }, function(resp){
      if (!resp || resp.error) { msg = 'Error: '+esc(resp && resp.error || 'unknown'); render(); return; }
      cnt = resp.n || (cnt+1);
      msg = esc(resp.msg) + ' &mdash; <span class="n">n = '+cnt+'</span>';
      render();
    });
  }
}
```

A two-zone pager (left half = prev, right half = next), as in the Ranking plugin:

```js
function __mupf_click(x, y){
  if (y < H - 52) return;                    // only the bottom strip is interactive
  if (x < W/2) { if (page > 0)              { page--; render(); } }   // left = prev
  else         { if (page < totalPages()-1) { page++; render(); } }   // right = next
}
```

Practical rules:

- `W`/`H` are your window size from the manifest `client.window` — hardcode them as globals and lay out against exact pixels (the window does **not** resize).
- Only `__mupf_click` is delivered. **There is no hover, mouse-move, drag, or key event to JS.** Design entirely around clicks (plus the manifest hotkey, which toggles the window).
- Since hit-testing is manual, keep clickable regions on simple rectangular bands (header, body rows of known height, a bottom button/pager). If you need per-row clicks, compute the row from `y` against your row height and the body's top offset.

### 2.4 The host-reserved title strip and close hotspot

The host reserves two areas of the window; everything else reaches `__mupf_click`:

- **Top ~38px = drag strip.** Clicking/dragging there moves the window. Draw your title bar in this band — it reads as a normal title bar and gives you free dragging.
- **Top-right ~38×38px = built-in close hotspot.** Clicking there closes the window even if your `__mupf_click` does nothing. It's good practice to *also* paint a `✕` glyph there so users see the affordance (and you may handle it yourself with `MUPF.close()` for windows shorter than 38px).

```js
// a 42px title bar that sits in the drag strip, with a painted close glyph top-right
'<div id="tb"><span id="tt"><img class="crown" src="crown.svg">Server Ranking</span>' +
  '<div id="x">&#10005;</div></div>'   // U+2715 ✕ renders fine
```

```css
#tb{position:relative;height:42px;display:flex;align-items:center;justify-content:center;
    background:linear-gradient(#8e2f2f,#561c1c);border-bottom:2px solid #b34a4a}
#x {position:absolute;top:0;right:0;width:42px;height:42px;display:flex;
    align-items:center;justify-content:center;color:#f3a6a6;font-size:16px;font-weight:bold}
```

### 2.5 Rendering lists, tables, forms and buttons (as strings)

You assemble markup with string concatenation. **Always escape user/DB-derived text** before inlining it.

**A data table (header + rows):**

```js
function rowsHtml(){
  var s = page*PAGE, slice = rows.slice(s, s+PAGE), h = '';
  for (var i=0;i<slice.length;i++){
    var r = slice[i];
    h += '<tr><td class="r">'+esc(r.rank)+'</td>'+
         '<td class="nm">'+esc(r.name)+'</td>'+
         '<td class="n">'+esc(r.reset)+'</td>'+
         '<td class="jl">'+esc(r.level)+'</td></tr>';
  }
  return h;
}
function tableHtml(){
  if (!rows.length) return '<div class="msg">No records.</div>';
  return '<table><thead><tr><th class="r">#</th><th>Player</th>'+
         '<th class="n">Reset</th><th class="jl">Level</th></tr></thead>'+
         '<tbody>'+rowsHtml()+'</tbody></table>';
}
```

**A "button":** there are no real `<button>` events — a button is just a styled `<div>` at a fixed position that you detect in `__mupf_click`:

```css
#btn{position:absolute;left:16px;right:16px;bottom:16px;height:44px;
     display:flex;align-items:center;justify-content:center;
     background:linear-gradient(#e8b54a,#c8922e);color:#2a1d05;
     font-size:14px;font-weight:700;letter-spacing:2px;border:1px solid #8a5a14}
```

```js
// markup
'<div id="btn">PING SERVER</div>'
// hit-test (button band at the bottom)
if (y >= H-60) { /* invoke server, then render() */ }
```

**A "form":** there are no text inputs. Model forms as state in globals that buttons mutate — e.g. a numeric stepper (▲/▼ regions adjust a global, a Submit region `MUPF.invoke`s it), or a toggle (a region flips a boolean, then `render()`). Draw the current value into the page each render; collect it by sending the global with `MUPF.invoke` when the user taps Submit.

### 2.6 CSS that works — and what doesn't

The engine is litehtml painting through GDI. A solid, modern look is achievable, but only with the supported subset.

**Supported and reliable:**

- `background-color` (solid) and **`linear-gradient`** backgrounds (2-stop, via GDI `GradientFill` — first/last stop, axis = dominant of start→end).
- `border` / `border-*` (solid only), `width`/`height`/`padding`/`margin`.
- `display:flex` (with `align-items`, `justify-content`, `gap`), `table` / `border-collapse`, `nth-child` (e.g. zebra rows).
- text: `color`, `font` shorthand, `font-size`/`font-weight`, `letter-spacing`, `text-transform`, `text-shadow`, `white-space:nowrap`, `vertical-align`.
- `position:absolute/relative`, `overflow:hidden`, `-webkit-user-select:none`.

**Not rendered — do not rely on these:**

| Feature | Why / workaround |
|---|---|
| `border-radius` (rounded corners) | not drawn — use square panels, or bake rounding into an SVG frame |
| `box-shadow` | not drawn — fake depth with a 1px border + a `linear-gradient` |
| `radial-gradient` / `conic-gradient` | not drawn (the host stubs them) — use `linear-gradient` |
| `:hover`, transitions, animations | no — there is no mouse-move event |
| `setTimeout` / `setInterval` / `fetch` | not available — re-render on click/reply only |
| `%` heights vs the viewport | unreliable — the window is fixed size; use **fixed px** |

> 💡 Because the window size is fixed and known (`W`/`H`), lay out against exact pixels. Bundled plugins use a `#wrap{height: H-… px}` panel with an absolutely-positioned header, body, and bottom strip — predictable and pixel-precise.

A working zebra table + gradient header (from the Ranking plugin):

```css
#wrap{position:relative;height:554px;background:linear-gradient(#1b1c22,#121319)}
table{width:100%;border-collapse:collapse}
th{text-align:left;padding:9px 8px;font-size:10px;text-transform:uppercase;
   letter-spacing:.8px;color:#7f8696;border-bottom:1px solid #2b2f3a;white-space:nowrap}
td{padding:8px;border-bottom:1px solid #1e2027;white-space:nowrap;vertical-align:middle}
tr:nth-child(even) td{background:#181a21}     /* zebra striping works */
```

### 2.7 Fonts and glyphs — beware tofu

Text is drawn with `TextOutW` and **has no font fallback**. A character the chosen font lacks renders as an empty box ("tofu"). The default family is Arial; bundled plugins use `"Segoe UI", Tahoma, sans-serif`.

- Safe glyphs in Segoe UI / Tahoma: `✕` (U+2715, `&#10005;`) and `★` render.
- **Triangle arrows `◀ ▶` (U+25C0 / U+25B6) are NOT in those fonts → they show as boxes.** Don't use them for prev/next — draw arrows as **SVG** instead.

```js
// safe close glyph
'<div id="x">&#10005;</div>'      // ✕
```

### 2.8 Images and SVG (the reliable way to get icons)

`<img>` and CSS `background-image` render via a vector rasterizer (**nanosvg**). **SVG only — PNG/JPG are not supported yet.**

```html
<img class="crown" src="crown.svg">
```

```css
.bdg { background-image:url(badge.svg); background-repeat:no-repeat;
       background-position:center; background-size:26px 26px; }
```

- Reference assets **relative** — `src="crown.svg"` resolves to your `client/crown.svg` (the host also tries the `client/` prefix automatically). Ship the `.svg` files in `client/` and list them in the manifest `client.assets`; they're packed with the plugin.
- nanosvg supports paths, `rect`/`circle`/`ellipse`/`line`/`polygon`, flat fills, and `linearGradient`. It does **not** render SVG `<text>` — put text in HTML *on top of* the SVG (the Ranking badges do exactly this: an SVG disc as `background-image`, the rank number as HTML text centered over it).
- SVG sidesteps both limitations above: it gives you crisp **rounded frames, gradient badges, and arrow icons** with none of the `border-radius`/glyph problems.

```css
/* rank badge: SVG disc behind, HTML number on top */
.bdg{display:inline-block;width:26px;height:26px;line-height:26px;text-align:center;
     font-weight:700;color:#dfe3ec;background-image:url(badge.svg);
     background-repeat:no-repeat;background-position:center;background-size:26px 26px}
.bdg.g{background-image:url(badge_gold.svg);color:#3c2a06}   /* per-rank variants */
```

### 2.9 Theming a window — a complete entry page

Putting it together: a fixed-size panel, a gradient title bar in the drag strip, a painted close glyph in the reserved top-right, a body, a bottom action strip, all state in globals, all input through `__mupf_click`. This is a full, copy-pasteable `client/index.html`:

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<style>
  html, body { margin:0; padding:0; background:#0e0f13; }
  #boot { padding:40px 0; text-align:center; color:#7f8696; font:13px "Segoe UI", Tahoma, sans-serif; }
</style>
</head>
<body>
  <div id="boot">Loading&#8230;</div>
  <script>
    var W = 420, H = 300;                         // from plugin.json client.window
    var msg = 'Click PING to call the server.', cnt = 0;   // state, persists across renders

    var CSS =
      'html,body{margin:0;padding:0;font:13px "Segoe UI",Tahoma,sans-serif;color:#d7dbe4;-webkit-user-select:none;user-select:none}' +
      'body{background:#0e0f13;border:1px solid #2b2f3a}' +
      '#wrap{position:relative;height:294px;background:linear-gradient(#1b1c22,#121319)}' +
      '#tb{position:relative;height:36px;display:flex;align-items:center;justify-content:center;background:linear-gradient(#8e2f2f,#561c1c);border-bottom:2px solid #b34a4a}' +
      '#tt{color:#f6e7e7;font-size:14px;font-weight:700;letter-spacing:1.5px;text-shadow:0 1px 2px #000}' +
      '#x{position:absolute;top:0;right:0;width:36px;height:36px;display:flex;align-items:center;justify-content:center;color:#f3a6a6;font-size:16px;font-weight:bold}' +
      '#bd{padding:22px 18px;text-align:center}' +
      '.msg{color:#cfd3dd;font-size:14px;line-height:1.5;min-height:48px}' +
      '.n{color:#ffd479;font-weight:700}' +
      '#btn{position:absolute;left:16px;right:16px;bottom:16px;height:44px;display:flex;align-items:center;justify-content:center;background:linear-gradient(#e8b54a,#c8922e);color:#2a1d05;font-size:14px;font-weight:700;letter-spacing:2px;border:1px solid #8a5a14}';

    function esc(s){ return String(s==null?'':s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }

    function view(){                              // a COMPLETE document, NO <script>
      return '<!DOCTYPE html><html><head><meta charset="utf-8"><style>'+CSS+'</style></head><body><div id="wrap">'+
        '<div id="tb"><span id="tt">MY PLUGIN</span><div id="x">&#10005;</div></div>'+   // ✕ in the reserved top-right
        '<div id="bd"><div class="msg">'+msg+'</div></div>'+
        '<div id="btn">PING SERVER</div>'+
        '</div></body></html>';
    }
    function render(){ MUPF.render(view()); }

    function __mupf_click(x, y){
      if (y < 36 && x >= W-38) { MUPF.close(); return; }     // top-right close
      if (y >= H-60) {                                        // bottom button band
        MUPF.invoke('ping', { n: cnt }, function(resp){
          if (!resp || resp.error) { msg = 'Error: '+esc(resp && resp.error || 'unknown'); render(); return; }
          cnt = resp.n || (cnt+1);
          msg = esc(resp.msg) + ' &mdash; <span class="n">n = '+cnt+'</span>';
          render();
        });
      }
    }

    render();   // first paint (no server call needed to show the UI)
  </script>
</body>
</html>
```

For a **tier-1 launcher** (`client/launcher.html`), the whole tiny window is the click target and the click just opens the main window — same engine, same rules, no `MUPF.render` needed:

```html
<!DOCTYPE html><html><head><meta charset="utf-8">
<style>
  html,body{margin:0;padding:0;-webkit-user-select:none;user-select:none}
  #b{box-sizing:border-box;width:146px;height:42px;display:flex;align-items:center;justify-content:center;
     background:linear-gradient(#23242e,#14151c);border:2px solid #c8922e;
     color:#f3e6c8;font:13px "Segoe UI",Tahoma,sans-serif;font-weight:700;letter-spacing:2px;text-shadow:0 1px 2px #000}
  #dv{color:#ffd479;margin-right:9px;letter-spacing:1px}
</style></head>
<body>
  <div id="b"><span id="dv">DV</span>TEST</div>
  <script>
    function __mupf_click(x, y){ MUPF.open(); }   // open this plugin's main window
  </script>
</body></html>
```

> 💡 Theming recipe that survives the CSS limits: a 1px `border` + a vertical `linear-gradient` panel for "depth" (no `box-shadow`), a contrasting gradient header bar in the drag strip, square panels (no `border-radius`), SVG for any rounded/icon art, and a fixed-px layout with an absolutely-positioned bottom strip for actions or paging. The bundled `com.dvteam.ranking` and `com.dvteam.test2tier` plugins are complete, working references for this exact structure.

See [MUPF Server API](MUPF-Server-API.md) for the matching `OnInvoke`/`ctx:reply` handler your `MUPF.invoke` calls reach, and [MUPF Plugins](MUPF-Plugins.md) for the manifest, capabilities, packaging, and the 2-tier launcher fields.

---

## 3. The server side (`server.lua`)

The server half of a plugin is one file — `server/server.lua` — loaded once into the GameServer's Lua state at startup (or on `Reload Script`). It runs in a **fresh per-plugin environment**: your writes stay local, your reads fall through to the trusted server `_G`. It **never leaves the server**, so this is where DB access and player reads belong. This section is the practical how-to; for the per-call reference see [MUPF Server API](MUPF-Server-API.md).

### 3.1 The shape of every server.lua

Three things, always in this order: build a hooks table, define `OnInvoke` (and optionally `OnLoad`), then `PluginRegister` it at the very end.

```lua
local Plugin = {}

function Plugin.OnLoad()
    host.log("my plugin ready")          -- optional; runs once after load
end

function Plugin.OnInvoke(ctx, fn, args, reqId)
    -- fn/args are UNTRUSTED. Validate, then reply EXACTLY once per reqId.
    if fn == "ping" then
        ctx:reply(reqId, { pong = true, you = ctx:playerName() })
        return
    end
    ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
end

PluginRegister("com.yourteam.yourplugin", Plugin)   -- id MUST equal plugin.json "id"
return Plugin
```

⚠️ `server.lua` is loaded **once**. Editing it requires a server restart or the GameServer-window **`Reload Script`** command — relog only refreshes the *client* side. (See [MUPF Plugins → Dev vs ship](MUPF-Plugins.md#developing-a-plugin-dev-vs-ship).)

### 3.2 `PluginRegister(idStr, hooks)`

Call it **once**, at the end of the file. `idStr` must be byte-for-byte equal to the manifest `id` — registration is rejected otherwise, and the plugin gets no invocations:

```lua
PluginRegister("com.dvteam.ranking", Ranking)
```

🔒 The host only accepts `PluginRegister` from the plugin whose `server.lua` is *currently executing*. One plugin cannot register hooks for another id — a mismatched or stray id is logged and dropped. A plugin that never calls `PluginRegister` (or registers a table with no `OnInvoke`) is loaded but inert: its clients get a `no_handler` / `no_oninvoke` error.

| Hook | When it fires |
|---|---|
| `OnLoad()` | once, right after the chunk runs (optional) |
| `OnInvoke(ctx, fn, args, reqId)` | a client `MUPF.invoke(fn, args, cb)` arrived |

### 3.3 `OnLoad()`

Runs once, right after `server.lua` finishes loading. Use it for cheap, synchronous setup or a heartbeat log — nothing player-specific exists yet, so there is no `ctx`. Errors here are caught and logged; the plugin still loads.

```lua
function Ranking.OnLoad()
    host.log("ranking server ready")
end
```

`host.log(...)` writes to the server log's `plugin` channel and needs no capability. It is your only debugging window into server-side plugin code — use it liberally.

### 3.4 `OnInvoke(ctx, fn, args, reqId)` — the request entry point

Every `MUPF.invoke(fn, args, cb)` from the client lands here.

| Param | Type | Notes |
|---|---|---|
| `ctx` | userdata-like table | scoped to the **calling player** and **this request**; methods below |
| `fn` | string | the function name the client asked for — **untrusted** |
| `args` | Lua table | the client's payload, already parsed from JSON — **untrusted** |
| `reqId` | number | opaque request id; pass it back to `ctx:reply` |

The contract: **dispatch on `fn`, validate `args`, do work, reply once.** If `OnInvoke` raises a Lua error, the host catches it and sends the client `{ "error": "internal", "code": "INTERNAL" }` automatically — but you should prefer explicit error replies (§3.9) so the client gets a meaningful code.

```lua
function Plugin.OnInvoke(ctx, fn, args, reqId)
    if fn == "getProfile" then
        ctx:reply(reqId, { name = ctx:playerName(), level = ctx:playerLevel() })
        return
    end
    if fn == "ping" then
        local n = tonumber(args and args.n) or 0          -- coerce, never trust
        ctx:reply(reqId, { ok = true, n = n + 1 })
        return
    end
    ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })   -- default deny
end
```

### 3.5 The `ctx` object — full method reference

`ctx` is built per-request and carries the caller's player slot and the plugin id internally. Methods marked with a capability raise a Lua error (caught + logged, plugin keeps running) when that capability is **not** granted in *both* the manifest `permissions` and `policy.json` (see [Capabilities](MUPF-Plugins.md#capabilities-permissions)).

| Method | Capability | Returns / does |
|---|---|---|
| `ctx:aIndex()` | — (always) | the caller's in-world object index (player slot), a number |
| `ctx:playerName()` | `player.readBasic` | the caller's DB name; `nil` if logged out mid-request |
| `ctx:playerLevel()` | `player.readBasic` | the caller's level; `0` if logged out mid-request |
| `ctx:reply(reqId, tbl)` | — (always) | sends the response; call **once** per `reqId` |
| `ctx:sql(sql, params, cb)` | `db.query` | async parameterized query; `cb(rows)` fires later |
| `ctx:push(channel, tbl)` | `net.serverPush` | server→client push (delivery being finalized) |
| `ctx:open()` | `ui.window` | ask the client to open this plugin's window (being finalized) |

💡 The player may log out between sending the request and your handler (or SQL callback) running. `ctx:playerName()` returns `nil` and `ctx:playerLevel()` returns `0` in that case — guard for it rather than assuming a live player:

```lua
local name = ctx:playerName()
if not name then
    ctx:reply(reqId, { error = "gone", code = "BAD_REQUEST" })
    return
end
```

### 3.6 `ctx:reply(reqId, table)` — answering the request

`table` is serialized to JSON and delivered to the client's `MUPF.invoke` callback. The body **must be a table** (or `nil`/omitted → an empty object `{}`); anything else is a Lua error. You choose the shape — there is no enforced envelope. Reply exactly once: a second `reply` for the same `reqId` is a second wire message, not an update.

```lua
ctx:reply(reqId, { rows = out, total = #out })
```

⚠️ With an async `ctx:sql`, `OnInvoke` returns **before** the data is ready — the reply happens later, **inside the SQL callback** (§3.7). Do not also reply at the top level, or the client sees two messages for one request.

### 3.7 `ctx:sql(sql, params, cb)` — parameterized async SQL

The game loop is **never blocked**: the query runs on a worker and `cb(rows)` fires back on the Lua thread when the result arrives. Two iron rules:

1. **Never concatenate user input into SQL.** Use `?` placeholders; the host substitutes each `params` entry by position.
2. **Reply from inside `cb`.** That is the only place the data exists.

```lua
ctx:sql(
    "SELECT name, reset, level FROM character_info WHERE authority = ? ORDER BY reset DESC LIMIT 100",
    { 0 },                                  -- one param for the one '?'
    function(rows)
        local out = {}
        if rows then
            for i, r in ipairs(rows) do      -- rows is a 1-based array of row tables
                out[i] = { rank = i, name = r.name, reset = tonumber(r.reset) or 0 }
            end
        end
        ctx:reply(reqId, { rows = out })     -- reply here, not at top level
    end)
```

**How `?` substitution is typed.** The host walks the SQL and replaces each `?` with the matching `params[i]`, applying type-correct encoding:

| Lua param type | Becomes in SQL | Notes |
|---|---|---|
| number (integer) | `123` | bare, no quotes |
| number (float) | `1.5` round-trip-safe | non-finite (`NaN`/`inf`) → Lua error, query aborts |
| string | `'…'` escaped | passed through the host's `EscapeString`; **strings are the only quoted/escaped form** |
| boolean | `1` / `0` | |
| anything else (nil, table, function) | — | Lua error: `unsupported param type for '?'` |

⚠️ The `?` count and the `params` array length must line up. More `?` than params → `ctx:sql: more '?' than params` (Lua error). Extra params are simply unused. Pass `{}` (or omit by passing an empty table) when the query has no placeholders.

**The `rows` shape.** A `SELECT` that returns rows yields a **1-based array of row tables**, each keyed by column name (`r.name`, `r.reset`, …); column values arrive as strings, so coerce with `tonumber(...)` where you need numbers. A query with **no result rows** (or a non-`SELECT`) does **not** give you an array — guard with `if rows then` before iterating, exactly as the examples do. This is the same async-SQL model documented in [Database Structures](Database-Structures.md).

🔒 Each callback is routed by a **host-allocated opaque label**, never a plugin-chosen string — one plugin can never receive another plugin's SQL result. If the host cannot even spawn the query (resource exhaustion), `ctx:sql` raises a Lua error rather than silently dropping the request.

💡 **Capability tip:** `ctx:sql` requires `db.query`; calling it without the grant raises a Lua error that aborts the handler. Keep `db.query` out of your manifest entirely if the plugin does not need SQL.

### 3.8 Paging: server caps, client pages over the rows

The framework's intended pattern is **server returns up to N rows in one reply; the client paginates over those rows.** The server does *not* run a separate query per page. The Ranking plugin is the canonical example: it always returns the full top-100 and validates/echoes the requested `page` only so a malformed value can never reach the DB or break the client.

```lua
local Ranking = {}
local PAGE_SIZE = 10     -- must match the client's page size
local MAX_ROWS  = 100    -- the hard cap the server returns in one reply

local RANK_SQL =
    "SELECT guid, name, race, reset, level, level_master, level_majestic " ..
    "FROM character_info " ..
    "WHERE authority = 0 " ..
    "ORDER BY `reset` DESC, `level_majestic` DESC, `level_master` DESC, `level` DESC " ..
    "LIMIT 100"

function Ranking.OnInvoke(ctx, fn, args, reqId)
    if fn ~= "getRanks" then
        ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
        return
    end

    -- Sanitize the UNTRUSTED page index. Non-numbers, NaN, negatives,
    -- fractions and overflow all clamp into 0..maxPage.
    local page = tonumber(args and args.page) or 0
    if type(page) ~= "number" or page ~= page then page = 0 end   -- reject NaN
    page = math.floor(page)
    if page < 0 then page = 0 end
    local maxPage = math.floor((MAX_ROWS - 1) / PAGE_SIZE)        -- 0..9
    if page > maxPage then page = maxPage end

    ctx:sql(RANK_SQL, {}, function(rows)
        local out = {}
        if rows then
            for i, r in ipairs(rows) do
                if i > MAX_ROWS then break end
                out[#out + 1] = {
                    rank  = i,
                    name  = r.name or "",
                    class = tonumber(r.race) or 0,        -- RAW id; client maps id -> name
                    reset = tonumber(r.reset) or 0,
                    level = tonumber(r.level) or 0,
                    ml    = tonumber(r.level_master) or 0,
                    jl    = tonumber(r.level_majestic) or 0,
                }
            end
        end
        ctx:reply(reqId, {
            page     = page,         -- sanitized/echoed; informational only
            pageSize = PAGE_SIZE,
            total    = #out,         -- ROW COUNT (<=100), not page count
            rows     = out,
        })
    end)
end

PluginRegister("com.dvteam.ranking", Ranking)
return Ranking
```

The matching client (`client/index.html`) calls `MUPF.invoke("getRanks", { page: 0 }, cb)` once, caches `rows`, and slices `PAGE_SIZE` at a time in its `view()` — see [MUPF Client API](MUPF-Client-API.md). `total` here is the **row count** (≤ 100), not a page count; the client derives page count from `total / pageSize`.

💡 If your dataset is genuinely huge (won't fit one bounded reply), do server-side paging instead: pass a validated `page`/`limit` in `args` and bake `LIMIT ? OFFSET ?` into the SQL with `?` params. Clamp both before the query — never forward a raw client `limit`.

### 3.9 Validating untrusted `fn` and `args`

🔒 **`pluginId` routing is not authorization.** The host routes a request to your plugin by id; it does **not** vouch for `fn` or `args`. A modified client can send **any** `fn` and **any** `args` to **any** plugin it can reach. Treat both as hostile input on every call.

Concrete rules that the bundled plugins follow:

- **Whitelist `fn`.** Dispatch with explicit `if fn == "x"` branches and a default-deny `else` that replies `unknown_fn`. Never `loadstring`, index a function table by raw `fn`, or otherwise let `fn` choose code.
- **Coerce and clamp every `args` field.** `tonumber(args and args.page)` then floor/clamp; default strings with `args.foo or ""`. Reject `NaN` (`v ~= v`) and out-of-range values explicitly — see the Ranking `page` clamp above.
- **Never interpolate `args` into SQL.** Always go through `?` placeholders (§3.7). The host escapes for you; string concatenation re-opens injection.
- **`args` may be missing or the wrong type.** Guard with `args and args.field` — a client can omit the payload entirely (the host hands you an empty table on unparseable JSON).

```lua
local n = tonumber(args and args.n) or 0     -- test2tier: coerce, default, never trust
ctx:reply(reqId, { ok = true, n = n + 1 })
```

### 3.10 JSON ↔ Lua mapping

The conversion is automatic in both directions; know the rules so your payloads round-trip cleanly.

**Inbound (`args`):** JSON object → Lua table (string keys), JSON array → 1-based Lua table, numbers/strings/booleans → Lua values, `null` → `nil`.

**Outbound (`ctx:reply` / `ctx:push`):** the table is serialized back to JSON with this array-vs-object rule:

| Your Lua table | Serializes as |
|---|---|
| empty `{}` | JSON **array** `[]` (Lua can't distinguish empty array from empty object) |
| keys are exactly `1..n` (contiguous ints) | JSON array |
| any string key, or a gap/0/negative key | JSON object |

💡 Because an empty `{}` always becomes `[]`, an empty collection like `rows = {}` is the right idiom (the client gets `[]`). If you specifically need an *empty object*, give the table at least one keyed field. Numbers that are integral serialize without a decimal point; non-integral as floats.

⚠️ **Depth-32 limit.** Nesting deeper than 32 levels is truncated to `null` at that point (a stack-bounding safeguard). Keep replies flat-ish — a list of flat row tables, like Ranking's `rows`, is ideal. This applies to `args` parsing as well.

### 3.11 Error replies

There is no built-in error envelope — pick a shape and use it consistently. The bundled plugins use `{ error = "<reason>", code = "<CODE>" }`, mirroring the host's own auto-generated failures (`no_handler` / `NO_HANDLER`, `no_oninvoke` / `NO_HANDLER`, `internal` / `INTERNAL`). Reuse that convention so the client can branch on `resp.error`:

```lua
ctx:reply(reqId, { error = "unknown_fn",  code = "BAD_REQUEST" })   -- bad fn
ctx:reply(reqId, { error = "bad_input",   code = "BAD_REQUEST" })   -- failed validation
ctx:reply(reqId, { error = "gone",        code = "BAD_REQUEST" })   -- player logged out
```

The client checks `if (resp.error) { … }` first (see [MUPF Client API → `MUPF.invoke`](MUPF-Client-API.md#mupfinvokefn-args-cb--reqid)). Always reply on the error path too — a request with no reply leaves the client's callback hanging until it counts against `limits.maxPending`. If your handler throws instead, the host sends `{ error: "internal", code: "INTERNAL" }` as a backstop, but an explicit, specific code is far more useful to the UI.

---

## 4. The 2-tier launcher (always-on button)

A hotkey toggles a window, but you can't bind every plugin to a key — and players won't discover a hotkey they were never told about. A **tier-1 launcher** solves both: a small, always-visible button anchored to the game window that opens the plugin's main window on click. It is just a second HTML page belonging to the same plugin — same engine, same `__mupf_click` model, same rules as [the main window](MUPF-Client-API.md).

```
[always-on launcher button]  --click → MUPF.open()-->  [main window opens]
   client/launcher.html                                   client/index.html
       (tier 1)                                               (tier 2)
```

The bundled **`com.dvteam.test2tier`** plugin is the minimal working example; everything below is taken from it.

### The `client.launcher` manifest block

Declare the launcher inside `client`, alongside `window`:

```json
{
  "manifest": 1,
  "id": "com.dvteam.test2tier",
  "name": "2-Tier Test",
  "version": "1.0.0",
  "apiVersion": 1,
  "minApiVersion": 1,
  "author": "DV Team",
  "description": "Minimal 2-tier plugin: an always-on launcher button that opens a main window.",
  "server": { "entry": "server/server.lua" },
  "client": {
    "entry": "client/client.lua",
    "html":  "client/index.html",
    "launcher": {
      "html": "client/launcher.html",
      "width": 150, "height": 46,
      "anchor": "top-left", "x": 16, "y": 96
    },
    "window": { "html": "client/index.html", "width": 420, "height": 300, "resizable": false, "title": "2-Tier Test" }
  },
  "entryPoints": {
    "hotkey": "F6",
    "uiButton": { "icon": null, "label": "Test", "tooltip": "2-tier test" }
  },
  "permissions": ["ui.window", "ui.hotkey", "net.request"],
  "limits": { "maxAssetBytes": 65536, "maxMsgBytes": 8000, "maxPending": 16 }
}
```

| Field | Meaning |
|---|---|
| `html` | the launcher button's own HTML page (e.g. `client/launcher.html`). A separate page from `client.html` / `window.html`. |
| `width` / `height` | the launcher window size in px. Keep it small — it's a pill/button, not a panel. |
| `anchor` | which corner/edge of the **game window** the launcher is pinned to (enum below). |
| `x` / `y` | pixel offset from that anchor, measured **into** the game client area. |

**Anchor enum:** `top-left` · `top-right` · `top-center` · `bottom-left` · `bottom-right` · `bottom-center` · `left` · `right` · `center`.

Position is **relative to the game window**, not the desktop, so it lands in the same place at any resolution, follows the game window when it moves, and hides when you alt-tab away (returning when the game regains focus). With `"anchor": "top-left", "x": 16, "y": 96`, the launcher sits 16px right and 96px down from the top-left of the game client area.

> 🔒 The launcher needs the **`ui.window`** capability (it's how `MUPF.open()` is allowed to open a window) — and that cap must be granted in `Data/ClientLuaPlugin/policy.json`, not just requested in the manifest. Without it the click is a no-op. See [Capabilities](MUPF-Plugins.md#capabilities-permissions).

### Always-on / fixed / non-movable / non-closable

The host treats the launcher window differently from a main window. It is:

- **always visible** while you're in-game (you don't open it; it's just there),
- **fixed in place** at its `anchor` + `x`/`y` — there is no drag strip,
- **non-movable** and **non-closable** — the top-right close hotspot and the drag strip that a main window has do **not** apply here.

You get none of that behavior to manage yourself — it's enforced by the host purely from the `client.launcher` manifest block. Your `launcher.html` only has to draw the button and react to a click.

> ⚠️ Because the launcher can't be closed or dragged, treat its **whole surface as the click target** and don't paint a fake title bar or "✕" on it — there's nothing to close. Reserve those affordances for the tier-2 window.

### `launcher.html` — open the main window with `MUPF.open()`

The launcher page is an ordinary client page: it has no live DOM, so build static markup and handle clicks via the global `__mupf_click(x, y)`. The whole button is the hotspot, so you can ignore the coordinates and just call `MUPF.open()`:

```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<style>
  html, body { margin:0; padding:0; -webkit-user-select:none; user-select:none; }
  /* Tier-1 launcher: a small always-on button. The whole window is the click target;
     a click opens the plugin's main window via MUPF.open(). Fixed/non-movable/non-closable
     window behavior is handled by the host (position comes from plugin.json client.launcher). */
  #b {
    box-sizing:border-box; width:146px; height:42px;
    display:flex; align-items:center; justify-content:center;
    background:linear-gradient(#23242e, #14151c);
    border:2px solid #c8922e;
    color:#f3e6c8; font:13px "Segoe UI", Tahoma, sans-serif; font-weight:700; letter-spacing:2px;
    text-shadow:0 1px 2px #000;
  }
  #dv { color:#ffd479; margin-right:9px; letter-spacing:1px; }
</style>
</head>
<body>
  <div id="b"><span id="dv">DV</span>TEST</div>
  <script>
    // host forwards a click here (client px). The whole button opens the main window.
    function __mupf_click(x, y) { MUPF.open(); }
  </script>
</body>
</html>
```

`MUPF.open()` with no argument opens **this** plugin's main window (`client.window`). See [`MUPF.open([id])`](MUPF-Client-API.md#mupfopenid--tier-1-launcher).

> 💡 The launcher button is `150×46` in the manifest but the visible `#b` div is `146×42` — the 2px inset leaves room for the gold border without clipping. The same CSS rules as everywhere else apply: solid fills, `linear-gradient`, `flex`, borders, text — **no** `border-radius`, `box-shadow`, hover, or transitions (see [CSS support](MUPF-Client-API.md#css-support-litehtml--gdi)). For a rounded pill or an icon, bake it into an SVG.

### Opening *another* plugin from a launcher

`MUPF.open` takes an optional plugin id. Passing one opens **that** plugin's main window instead of your own — useful for a shared "hub" launcher that fronts several plugins:

```js
function __mupf_click(x, y) {
  // a two-button launcher: left half opens this plugin, right half opens another
  if (x < 75) MUPF.open();                       // this plugin's main window
  else        MUPF.open("com.dvteam.ranking");   // another plugin's main window
}
```

The target plugin must itself be installed and have `ui.window`; otherwise the call is a no-op (`__mupf_open` simply finds no window to open).

### Wiring the main window back to the server

Opening the window is tier 1's only job. The tier-2 page does the real work — it talks to `server/server.lua` over the request/reply bridge exactly like a single-tier plugin. The `test2tier` main window pings the server on click:

```js
function __mupf_click(x, y){
  if (y < 36 && x >= W-38) { MUPF.close(); return; }   // top-right = close (tier 2 IS closable)
  if (y >= H-60) {                                      // bottom button = ping
    MUPF.invoke('ping', { n: cnt }, function(resp){
      if (!resp || resp.error) { msg = 'Error: '+esc(resp && resp.error || 'unknown'); render(); return; }
      cnt = resp.n || (cnt+1);
      msg = esc(resp.msg) + ' &mdash; <span class="n">n = '+cnt+'</span>';
      render();
    });
  }
}
```

```lua
function P.OnInvoke(ctx, fn, args, reqId)
    if fn == "ping" then
        local n = tonumber(args and args.n) or 0
        ctx:reply(reqId, { ok = true, msg = "pong from server", n = n + 1 })
        return
    end
    ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
end
```

Note the contrast: the launcher (`client/launcher.html`) only calls `MUPF.open()` and is non-closable; the main window (`client/index.html`) hit-tests its own title bar to call `MUPF.close()` and uses `MUPF.invoke` for server calls. Two pages, one plugin, one `server.lua`. See the [Server API](MUPF-Server-API.md) for `OnInvoke` / `ctx:reply`.

### When to use a launcher vs a hotkey vs both

| You want… | Use |
|---|---|
| A discoverable, always-visible entry point (most players never learn hotkeys) | **launcher** (`client.launcher`) |
| A fast toggle for power users, costs no screen space | **hotkey** (`entryPoints.hotkey`) |
| Both — visible button **and** a quick key | **both** (the `test2tier` example ships `client.launcher` + `"hotkey": "F6"`) |
| A single "hub" button that opens several plugins | one launcher calling `MUPF.open("com.x.y")` per region |
| A panel that only the server should pop (e.g. on an event) | neither — let the server open it; you can still keep a hotkey/launcher as a manual fallback |

A launcher and a hotkey are not exclusive — both end up opening the **same** main window, and the server can open it too. The main window opens, drags, and closes normally regardless of how it was opened.

> 💡 Adding a launcher costs nothing on the server side: it's purely a `client.launcher` block plus one `launcher.html`. The `server.lua` doesn't change — tier 1 never calls the server, it only calls `MUPF.open()`.

See also: [MUPF Plugins → 2-tier plugins](MUPF-Plugins.md#2-tier-plugins--an-always-on-launcher-button), [MUPF Client API](MUPF-Client-API.md), [MUPF Server API](MUPF-Server-API.md).

---

## 5. Packaging, shipping & operations

A MUPF plugin has two lives: a fast **dev loop** where you edit a source folder and relog, and a **shipped** form where the whole plugin is one encrypted `.mupf` file dropped into the plugins folder. This section is the practical how-to for both, plus the operator-side `policy.json` and runtime reload. For the conceptual overview, see [MUPF Plugins → Developing a plugin](MUPF-Plugins.md#developing-a-plugin-dev-vs-ship).

> The GameServer reads **both** entry types from the same folder — source **folders** (dev) and **`.mupf` files** (shipped) — and you can mix them. Everything lives under `Data/ClientLuaPlugin/`.

---

### 5.1 Dev loop — work from a source folder

During development you do **not** pack. Keep the plugin as a plain folder under the plugins directory and let the server re-read it on each connect.

```
Data/ClientLuaPlugin/
├─ policy.json                 ← operator grants (see 5.4)
└─ com.yourteam.yourplugin/    ← your plugin, as a folder
   ├─ plugin.json
   ├─ server/server.lua
   └─ client/index.html  (+ client.lua, *.svg …)
```

This relies on `MUPF_DEV_HOTRELOAD` being `1` (the default in dev builds, set in `Game/PluginMgr.cpp`). With it on, every client connect re-packs the plugin from disk before streaming it, so:

| You edited… | To see the change |
|---|---|
| `client/index.html`, CSS, JS, an `.svg`, or `plugin.json` (window size, hotkey, label, launcher) | **relog** — the client files are re-zipped and re-streamed on the next connect |
| `server/server.lua` (or you added / removed a plugin) | **restart the GameServer**, or use the GS-window **Reload Script** (see 5.3) |

💡 The split is structural, not a limitation you can configure away: client files are streamed to the player at connect (so a relog re-sends them), but `server.lua` is loaded **once** into the shared Lua stack at startup. Editing it on disk changes nothing until that stack reloads.

⚠️ Hot-reload re-packs **every** plugin on **every** login. That is fine for one developer; leave it on only in dev. See 5.5.

---

### 5.2 Ship — pack into one `.mupf` with MupfPacker

When the plugin is ready, pack the **folder** into a single encrypted file with `MupfPacker`. It links the same cipher as the server and client (`Common/MupfPack.cpp`), so the key exists in exactly one place. See [`tools/MupfPacker/README.md`](../../tools/MupfPacker/README.md).

```bat
:: <plugin_folder> [out.mupf]   — out defaults to <foldername>.mupf in the cwd
MupfPacker.exe Data\ClientLuaPlugin\com.dvteam.ranking
:: -> com.dvteam.ranking.mupf   (whole plugin, encrypted)
```

Then ship and install it:

1. Delete (or move out) the source **folder** for that plugin from `Data/ClientLuaPlugin/` so it is not loaded twice.
2. Drop the `.mupf` into `Data/ClientLuaPlugin/`.
3. Grant its capabilities in `policy.json` under the same `id` (5.4).
4. Restart the GameServer (or **Reload Script**) so `server.lua` is loaded from the package.

🔒 Build `MupfPacker.exe` as a C++ exe and protect it with **Themida** (like the client DLL). The secret is compiled in; a plaintext/script packer would leak the key and let anyone forge or decrypt packages — that is the whole reason the packer is a compiled binary, not a loose script.

---

### 5.3 Runtime reload (no GS restart) — the GS-window *Reload Script*

The GameServer window's **`Reload Script`** command reloads plugins **without restarting the process**. Internally it re-scans `Data/ClientLuaPlugin/` (folders **and** `*.mupf`), re-packs each plugin, and reloads every plugin's `server.lua` into the Lua stack (`PluginMgr::Reload()`).

| Side | When the reload takes effect |
|---|---|
| **Server** — `server.lua`, manifests, the packed client blobs | **immediately**, server-wide |
| **Client** — the in-game UI | players already online keep the old UI until they **relog**; new logins get the new version right away |

Use it after editing `server.lua`, after dropping in a new `.mupf`, or after adding/removing a plugin — it avoids a full restart. The client side still follows the connect-time streaming rule, so tell online testers to relog.

---

### 5.4 Capabilities — `policy.json` (manifest ∩ policy, deny-by-default)

A plugin's effective capabilities are the **intersection** of what its manifest `permissions` array requests and what the operator grants in `Data/ClientLuaPlugin/policy.json`, keyed by plugin `id`. Anything not in **both** is denied. A plugin not listed in `policy.json` — or running with no `policy.json` at all — gets **nothing**.

```json
// Data/ClientLuaPlugin/policy.json
{
  "com.dvteam.ranking":  ["ui.window", "ui.hotkey", "net.request", "db.query"],
  "com.dvteam.test2tier": ["ui.window", "ui.hotkey", "net.request"]
}
```

The manifest that pairs with the first entry requests exactly those caps, so all four are granted:

```json
"permissions": ["ui.window", "ui.hotkey", "net.request", "db.query"]
```

How the intersection plays out:

| Requested in `plugin.json` | Listed in `policy.json` | Result |
|---|---|---|
| `db.query` | `db.query` | **granted** — `ctx:sql(...)` works |
| `db.query` | *(absent)* | denied — logged as `requested but NOT granted by policy.json`; `ctx:sql` raises a Lua error (caught, the call fails, plugin keeps running) |
| *(not requested)* | `net.serverPush` | not granted — a cap must be requested **and** allowed |

For the full capability list and which `ctx:*` method each one gates, see [MUPF Plugins → Capabilities](MUPF-Plugins.md#capabilities-permissions) and [MUPF Server API](MUPF-Server-API.md).

⚠️ Missing or malformed `policy.json` is treated as **deny-all** — the server logs `no policy.json … all caps DENIED` and every plugin runs with zero capabilities. A new plugin is invisible until you both ship it and add its `id` here.

💡 In dev, grant the caps you are testing. In production, grant **only** what each plugin actually needs — narrowing `policy.json` is the operator's lever even when the manifest asks for more.

---

### 5.5 The `.mupf` format & what reaches the client

A `.mupf` is the single distributable file for a plugin:

```
"MUPF" + version + XOR(secret, ZIP{ plugin.json, server/…, client/… })
```

- It is a ZIP of the **whole** plugin, then XOR-encrypted with the embedded secret.
- **Self-describing:** the loader accepts either this encrypted form or a plain `PK…` zip (dev folders), so the same decode path handles both.

On load, the server unpacks the `.mupf` in memory and splits it:

- **`server/server.lua`** is loaded **directly into the Lua stack** and runs server-side. **It is never sent to clients** — verify this by reading the loader: it re-packs only `plugin.json` + `client/*` for the wire (`kv.first == "plugin.json" || kv.first.rfind("client/", 0) == 0`), explicitly excluding `server/`.
- **Client files** (manifest + `client/*`) are re-zipped, encrypted, and streamed to each player at connect.

🔒 Put nothing secret in `client/` — those files are delivered to every player who logs in and can be inspected. Keys, queries, and any trust logic belong in `server.lua`, which stays on the server.

---

### 5.6 Operations checklist

| Setting | Dev | Production |
|---|---|---|
| `MUPF_DEV_HOTRELOAD` (`Game/PluginMgr.cpp`) | `1` — relog picks up client edits | **`0`** — cached blob, no per-login re-pack |
| Plugin form in `Data/ClientLuaPlugin/` | source **folder** | **`.mupf`** file |
| Client wire blob | plain `PK…` zip | encrypted (`MUPF` magic) |
| `policy.json` | grant the caps you test | grant **only** what each plugin needs |
| Apply a `server.lua` / manifest change | restart **or** *Reload Script* | restart **or** *Reload Script* (clients relog) |

```bat
:: production ship, end-to-end:
MupfPacker.exe Data\ClientLuaPlugin\com.yourteam.yourplugin
copy com.yourteam.yourplugin.mupf <server>\Data\ClientLuaPlugin\
:: 1) remove the source folder from the prod plugins dir (avoid double-load)
:: 2) add "com.yourteam.yourplugin": [ ...caps... ] to policy.json
:: 3) build the GameServer with MUPF_DEV_HOTRELOAD=0
:: 4) restart the GameServer (or use the GS-window "Reload Script")
```

⚠️ Set `MUPF_DEV_HOTRELOAD = 0` for production. Leaving it on re-packs every plugin on every login — harmless for one developer, wasteful at scale.

---

### See also

- **[MUPF Plugins](MUPF-Plugins.md)** — manifest fields, capabilities, the `.mupf` package, 2-tier launchers.
- **[MUPF Server API](MUPF-Server-API.md)** — `PluginRegister`, `OnInvoke`, the capability-gated `ctx:*` object.
- **[MUPF Client API](MUPF-Client-API.md)** — the in-game UI: `MUPF.invoke/render/close/open`, clicks, SVG.

---

## 6. Recipes — patterns for real plugins

A cookbook of complete, copy-pasteable patterns, ordered roughly by complexity. Every snippet follows the same model as the bundled plugins: **state in JS globals**, build a full HTML string, `MUPF.render()` it; on the server, **validate `fn`/`args`**, then `ctx:reply(reqId, …)`. Read the [Important model](MUPF-Client-API.md#-important-model--there-is-no-live-dom) section of the Client API first — there is no live DOM, no `setTimeout`, no `addEventListener`. For the full API surface see [MUPF Server API](MUPF-Server-API.md), [MUPF Client API](MUPF-Client-API.md), and [MUPF Plugins](MUPF-Plugins.md).

> 💡 Each recipe shows the *load-bearing* code. Wrap it in the boilerplate the bundled
> [Ranking](MUPF-Plugins.md) / [2-Tier Test](MUPF-Plugins.md#2-tier-plugins--an-always-on-launcher-button) plugins use: a `#boot` placeholder in `index.html`, an `esc()` HTML-escaper, a `CSS` string, a `frame(inner)` shell, `render()` → `MUPF.render(view())`.

---

### 6a. A paged list (N per page, prev/next pager)

This is exactly what the Ranking plugin does: the server returns the **whole** dataset in one reply (capped), and the **client** pages over it. The page index is a JS global, so prev/next just adjust it and re-render — **no extra server round-trip**.

The pager is a fixed strip along the bottom; `__mupf_click` hit-tests it by `y`, then splits left/right by `x`:

```js
var W = 640, H = 560, PAGE = 10;     // rows per page
var allRows = [], page = 0;          // <-- persist across MUPF.render

function totalPages(){ return Math.max(1, Math.ceil(allRows.length / PAGE)); }

function rowsHtml(){
  var s = page * PAGE, slice = allRows.slice(s, s + PAGE), h = '';
  for (var i = 0; i < slice.length; i++){
    var r = slice[i];
    h += '<tr><td>' + esc(r.rank) + '</td><td>' + esc(r.name) + '</td></tr>';
  }
  return h;
}

function frame(inner){
  return '<!DOCTYPE html><html><head><meta charset="utf-8"><style>' + CSS + '</style></head>'
    + '<body><div id="wrap">'
    +   '<div id="tb"><span id="tt">My List</span><div id="x">&#10005;</div></div>'
    +   '<div id="bd">' + inner + '</div>'
    +   '<div id="pg"><img class="pa" src="prev.svg">'
    +     '<span class="i">' + (page + 1) + ' / ' + totalPages() + '</span>'
    +     '<img class="pa" src="next.svg"></div>'
    + '</div></body></html>';
}
function pageHtml(){
  if (!allRows.length) return frame('<div class="msg">No rows.</div>');
  return frame('<table><tbody>' + rowsHtml() + '</tbody></table>');
}
function render(){ MUPF.render(pageHtml()); }

function __mupf_click(x, y){
  if (y < H - 52) return;                                   // not the pager strip
  if (x < W / 2){ if (page > 0)               { page--; render(); } }   // left half = prev
  else          { if (page < totalPages()-1)  { page++; render(); } }   // right half = next
}

MUPF.invoke('getRanks', {}, function(resp){                 // one fetch on load
  if (!resp || resp.error){ allRows = []; MUPF.render(frame('<div class="msg err">Error: '
    + esc(resp && resp.error || 'unknown') + '</div>')); return; }
  allRows = resp.rows || [];
  page = 0;
  render();
});
```

> ⚠️ Use SVG arrows (`prev.svg`/`next.svg`) for the pager, **not** `◀ ▶` glyphs — those characters are not in Segoe UI/Tahoma and render as empty boxes (tofu). See [Fonts & glyphs](MUPF-Client-API.md#fonts--glyphs). `✕` (U+2715, the `&#10005;` close cross) does render.

💡 **When to page on the server instead.** If the dataset is large (thousands of rows) or sensitive, send `{page: N}` to the server and `LIMIT`/`OFFSET` there. The client-side approach above is best when the full set is small and capped (the Ranking plugin caps at 100 rows, well under the 8000-byte payload limit — see [6f](#6f-performance--safety)).

---

### 6b. A form with fields + submit (server-side validation)

There are no `<input>` events, so a "form" is just **field state in globals** plus clickable widgets. The submit button calls `MUPF.invoke`; the server validates and replies with either an `ok` or an `error` you render inline.

**Client** — fields are globals; the widget rows are hit-tested by `y`-band:

```js
var W = 420, H = 300;
var form = { race: 0, agree: false };   // field state, persists across renders
var err = '';                           // last server validation message

var RACES = ['Dark Wizard', 'Dark Knight', 'Fairy Elf'];

function view(){
  var rows = '';
  for (var i = 0; i < RACES.length; i++)
    rows += '<div class="opt' + (form.race === i ? ' sel' : '') + '">' + esc(RACES[i]) + '</div>';
  return '<!DOCTYPE html><html><head><meta charset="utf-8"><style>' + CSS + '</style></head><body><div id="wrap">'
    + '<div id="tb"><span id="tt">Pick a class</span><div id="x">&#10005;</div></div>'
    + '<div id="bd">'
    +   '<div id="opts">' + rows + '</div>'
    +   '<div class="chk' + (form.agree ? ' on' : '') + '">' + (form.agree ? '[x]' : '[ ]') + ' I agree</div>'
    +   (err ? '<div class="err">' + esc(err) + '</div>' : '')
    + '</div>'
    + '<div id="btn">SUBMIT</div>'
    + '</div></body></html>';
}
function render(){ MUPF.render(view()); }

function __mupf_click(x, y){
  if (y < 36 && x >= W - 38){ MUPF.close(); return; }       // top-right = close
  // option rows live at ~48px each starting y=48
  if (y >= 48 && y < 48 + RACES.length * 30){ form.race = Math.floor((y - 48) / 30); render(); return; }
  if (y >= 48 + RACES.length * 30 && y < 48 + RACES.length * 30 + 30){ form.agree = !form.agree; render(); return; }
  if (y >= H - 60){                                         // SUBMIT
    err = '';
    MUPF.invoke('submit', { race: form.race, agree: form.agree }, function(resp){
      if (!resp || resp.error){ err = (resp && resp.error) || 'unknown'; render(); return; }
      MUPF.render(view().replace('id="opts"', 'id="opts" data-done')); // or render a success view
    });
  }
}
render();
```

**Server** (`server/server.lua`) — treat every field as hostile, validate ranges, reply with a clear error shape on failure:

```lua
function Plugin.OnInvoke(ctx, fn, args, reqId)
    if fn ~= "submit" then
        ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
        return
    end

    -- args is the parsed JSON (a Lua table). Validate before use.
    local race  = tonumber(args and args.race)
    local agree = args and args.agree == true
    if type(race) ~= "number" or race ~= race or race < 0 or race > 2 then  -- reject NaN / out of range
        ctx:reply(reqId, { error = "Pick a valid class.", code = "BAD_RACE" })
        return
    end
    if not agree then
        ctx:reply(reqId, { error = "You must agree first.", code = "NO_AGREE" })
        return
    end

    ctx:reply(reqId, { ok = true, race = math.floor(race) })
end
```

> ⚠️ The `error`/`code` shape is **your** convention — the host does not impose one. The client only sees `resp.error` when *the framework itself* fails (no handler, rate-limited, busy, parse error). For your own validation failures, return a normal table like `{ error = "...", code = "..." }` and check `resp.error` in the callback — exactly as the snippets above do.

---

### 6c. A shop-style buy flow (`ctx:sql` list → BUY → validate → re-render)

This composes 6a/6b with a database read. The server lists items via `ctx:sql`; each row carries an `id`; BUY sends `{id}`; the server re-validates the id against the live list, "buys", and replies. The client re-fetches (or patches its state) and re-renders.

**Server** — `db.query` capability required (declare `"db.query"` in `permissions` *and* grant it in `policy.json`):

```lua
local Shop = {}

local LIST_SQL = "SELECT id, name, price FROM plugin_shop_items WHERE enabled = 1 ORDER BY price ASC LIMIT 50"

function Shop.OnInvoke(ctx, fn, args, reqId)
    if fn == "list" then
        ctx:sql(LIST_SQL, {}, function(rows)
            local out = {}
            if rows then
                for i, r in ipairs(rows) do
                    out[#out + 1] = { id = tonumber(r.id) or 0, name = r.name or "", price = tonumber(r.price) or 0 }
                end
            end
            ctx:reply(reqId, { rows = out })   -- reply from INSIDE the SQL callback
        end)
        return
    end

    if fn == "buy" then
        local id = tonumber(args and args.id)
        if type(id) ~= "number" or id ~= id or id < 1 then           -- validate untrusted id
            ctx:reply(reqId, { error = "Bad item.", code = "BAD_ID" })
            return
        end
        -- Re-read the row by parameterized id; each ? is escaped/validated by the host (never concat).
        ctx:sql("SELECT id, name, price FROM plugin_shop_items WHERE id = ? AND enabled = 1 LIMIT 1",
            { math.floor(id) },
            function(rows)
                if not rows or not rows[1] then
                    ctx:reply(reqId, { error = "Item not available.", code = "GONE" })
                    return
                end
                local item = rows[1]
                -- ... apply the purchase server-side (deduct currency, grant item, etc.) ...
                ctx:reply(reqId, { ok = true, bought = item.name })
            end)
        return
    end

    ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
end

PluginRegister("com.yourteam.shop", Shop)
return Shop
```

**Client** — render a BUY hotspot per visible row, map the click `y` back to a row index, then re-list after a successful buy:

```js
var W = 480, H = 400, ROW_H = 40, TOP = 48, msg = '';
var items = [];

function rowsHtml(){
  var h = '';
  for (var i = 0; i < items.length; i++){
    var it = items[i];
    h += '<tr><td>' + esc(it.name) + '</td><td class="p">' + esc(it.price) + '</td>'
       + '<td class="buy">BUY</td></tr>';
  }
  return h;
}
function view(){
  return '<!DOCTYPE html><html><head><meta charset="utf-8"><style>' + CSS + '</style></head><body><div id="wrap">'
    + '<div id="tb"><span id="tt">Shop</span><div id="x">&#10005;</div></div>'
    + '<div id="bd"><table><tbody>' + rowsHtml() + '</tbody></table>'
    + (msg ? '<div class="msg">' + esc(msg) + '</div>' : '') + '</div>'
    + '</div></body></html>';
}
function render(){ MUPF.render(view()); }

function loadList(){
  MUPF.invoke('list', {}, function(resp){
    if (!resp || resp.error){ msg = 'Load failed: ' + esc(resp && resp.error || 'unknown'); render(); return; }
    items = resp.rows || []; render();
  });
}

function __mupf_click(x, y){
  if (y < 36 && x >= W - 38){ MUPF.close(); return; }
  if (x < W - 80) return;                          // BUY column is the right ~80px
  var idx = Math.floor((y - TOP) / ROW_H);         // which row was clicked
  if (idx < 0 || idx >= items.length) return;
  MUPF.invoke('buy', { id: items[idx].id }, function(resp){
    if (!resp || resp.error){ msg = esc(resp && resp.error || 'buy failed'); render(); return; }
    msg = 'Bought ' + esc(resp.bought) + '!';
    loadList();                                    // re-fetch so prices/stock are fresh, then re-render
  });
}

loadList();   // first paint after the list arrives
```

> 🔒 **Never trust the client's `id` (or `price`).** The server re-reads the row by id and applies the purchase from the *server's* numbers. The `pluginId` only routes the request to your handler — it does not authorize the contents (see [JSON ↔ Lua](MUPF-Server-API.md#json--lua)). Each `?` in `ctx:sql` is escaped/validated by the host, so the parameterized id is injection-safe; never build SQL by string concatenation.

---

### 6d. Loading & error states

Because the first paint happens *before* any reply arrives, give the user a visible state at every step: a boot placeholder in `index.html`, a rendered "loading" view, a rendered error view, and an "empty" view.

```js
var state = 'loading';   // 'loading' | 'ready' | 'error' | 'empty'
var rows = [], errMsg = '';

function view(){
  var inner;
  if (state === 'loading')      inner = '<div class="msg">Loading…</div>';
  else if (state === 'error')   inner = '<div class="msg err">Error: ' + esc(errMsg) + '</div>';
  else if (state === 'empty')   inner = '<div class="msg">Nothing here yet.</div>';
  else                          inner = renderRows();   // 'ready'
  return frame(inner);
}
function render(){ MUPF.render(view()); }

render();                                  // 1) paint "Loading…" immediately

MUPF.invoke('getData', {}, function(resp){ // 2) replace it when the reply lands
  if (!resp || resp.error){ state = 'error'; errMsg = (resp && resp.error) || 'unknown'; render(); return; }
  rows = resp.rows || [];
  state = rows.length ? 'ready' : 'empty';
  render();
});
```

| Framework failure (`resp.error`) | When it happens |
|---|---|
| `no_handler` / `no_oninvoke` | the plugin never called `PluginRegister`, or has no `OnInvoke` |
| `unknown_fn` *(your code)* | `fn` did not match any branch — return this yourself (see the skeletons) |
| `rate_limited` (`code:"RATE"`) | more than ~20 calls/second for this plugin (see [6f](#6f-performance--safety)) |
| `busy` | too many in-flight requests (the 64-request cap) |
| `internal` (`code:"INTERNAL"`) | your `OnInvoke` raised a Lua error (e.g. a `ctx:*` call without its capability) |
| `parse` | the reply JSON could not be parsed client-side |

> ⚠️ A request that never gets a reply is dropped after **10 seconds** (`REQUEST_TIMEOUT_MS`). If your handler can take a long DB path, make sure it always reaches a `ctx:reply` — including the not-found / error branches — or the user is stuck on "Loading…".

---

### 6e. A multi-window plugin (launcher + main + a second window)

A plugin can ship a **tier-1 launcher** (a small, always-on, fixed button) that opens the **main** window via `MUPF.open()`. With an id, `MUPF.open("com.x.y")` opens *another* plugin's main window. The launcher and main are just two HTML pages of one plugin — see [2-tier plugins](MUPF-Plugins.md#2-tier-plugins--an-always-on-launcher-button) and [MUPF.open](MUPF-Client-API.md#mupfopenid--tier-1-launcher).

**Manifest** — declare the launcher alongside the main window:

```json
"client": {
  "html":  "client/index.html",
  "launcher": {
    "html": "client/launcher.html",
    "width": 150, "height": 46,
    "anchor": "top-left", "x": 16, "y": 96
  },
  "window": { "html": "client/index.html", "width": 420, "height": 300, "resizable": false, "title": "My Plugin" }
}
```

**`client/launcher.html`** — the whole button is the click target; its click opens the main window:

```html
<!DOCTYPE html><html><head><meta charset="utf-8">
<style> html,body{margin:0;padding:0} #b{box-sizing:border-box;width:146px;height:42px;
  display:flex;align-items:center;justify-content:center;background:linear-gradient(#23242e,#14151c);
  border:2px solid #c8922e;color:#f3e6c8;font:13px "Segoe UI";font-weight:700;letter-spacing:2px} </style>
</head>
<body>
  <div id="b">OPEN</div>
  <script>
    function __mupf_click(x, y){ MUPF.open(); }   // no id = open THIS plugin's main window
  </script>
</body></html>
```

**Opening a second / sibling window** from the main page — pass an explicit plugin id (requires `ui.window`, and that target plugin must exist):

```js
function __mupf_click(x, y){
  if (y >= H - 60 && x < W / 2){ MUPF.open('com.dvteam.ranking'); return; } // open another plugin's window
  if (y >= H - 60)             { MUPF.close(); return; }                    // close this one
}
```

> ⚠️ The launcher window is **always visible, fixed, non-movable, and non-closable** — never call `MUPF.close()` from it (there is nothing to close). Its only job is `MUPF.open()`. The main window opens/closes/drags normally and can *also* be opened by the manifest `entryPoints.hotkey`.

💡 Both pages share the same engine and the same `__mupf_click(x, y)` model. There is **one JS context per window** — the launcher's globals are not the main window's globals; coordinate state through the server (`MUPF.invoke`) if two windows must agree.

---

### 6f. Performance & safety

Keep re-renders cheap and stay inside the host's caps. These are enforced — exceeding them yields an error reply or a dropped request, never a crash.

| Limit | Value | What happens past it |
|---|---|---|
| **Call rate** (per player, per plugin) | ~**20 / second** (1000 ms window) | reply `{ "error":"rate_limited", "code":"RATE" }` — socket stays open |
| **In-flight requests** (per window) | **64** pending | `MUPF.invoke` returns 0, callback gets `{ "error":"busy" }` |
| **Request timeout** | **10 s** | the pending callback is dropped (no reply ever arrives) |
| **Message payload** | **8000 bytes** (`limits.maxMsgBytes`) | the envelope fails to encode/decode → request/reply dropped |
| **`fn` / channel name** | **64 bytes** | over-long names rejected by the codec |
| **Per-asset bytes** | `limits.maxAssetBytes` (e.g. 65536) | oversized assets are not packed/streamed |
| **Total plugin blob** | **4 MiB** reassembly ceiling | the plugin is quarantined client-side |
| **JSON nesting** | depth **32** | deeper structures truncate to `null` — keep payloads flat-ish |

```lua
-- Server: keep replies under ~8 KB. Cap rows; don't echo huge blobs back.
ctx:sql(RANK_SQL, {}, function(rows)
    local out = {}
    if rows then
        for i, r in ipairs(rows) do
            if i > 100 then break end                 -- hard cap; 100 small rows ≈ a few KB
            out[#out + 1] = { rank = i, name = r.name or "" }
        end
    end
    ctx:reply(reqId, { rows = out })
end)
```

**Client: keep `MUPF.render()` cheap.** Each render re-parses the whole document and repaints, so:

- Don't re-fetch on every click if the data hasn't changed. In the paging recipe (6a) prev/next only mutate a global and re-render — **zero server calls** — which also keeps you well under the 20/s budget.
- Slice before you stringify (`allRows.slice(s, s + PAGE)`) — build HTML for the **visible** page only, not all 100 rows.
- Don't poll. There is no `setInterval`; if you need fresh data, fetch it on an explicit user action (a refresh button) so you control the rate.

**The SVG image cache works for you.** The host rasterizes each `.svg` once and caches it keyed by `src`, so referencing the same icon across many rows (the Ranking badge appears on every row) costs one decode, not one per render. A missing/invalid image is also cached as invalid, so a typo'd `src` won't re-parse every frame — but it *will* silently show nothing, so verify your asset names.

> ⚠️ Re-rendered views passed to `MUPF.render()` must contain **no `<script>`** — only the entry `index.html` carries the script. Your globals and functions stay alive between renders; the re-rendered HTML is pure markup. Keep CSS as a single string constant (`CSS`) and inject it once per `frame()`; don't rebuild it per row.

> 🔒 SVG only — **PNG/JPG are not supported**. Oversized SVGs are clamped to a 1024px raster. Bake rounded corners / shadows into SVG since `border-radius` and `box-shadow` are not drawn (see [CSS support](MUPF-Client-API.md#css-support-litehtml--gdi)).

---

## 7. Limits, gotchas & a pre-ship checklist

This is the "wish I'd known that earlier" section. The client UI is **not a browser** and the server bridge is **not a trusted RPC** — most plugin bugs come from assuming otherwise. Read every callout here before you ship. For the underlying API see [MUPF Client API](MUPF-Client-API.md), [MUPF Server API](MUPF-Server-API.md), and [MUPF Plugins](MUPF-Plugins.md).

### 7.1 The client is litehtml + duktape, not a browser

The UI is drawn by **litehtml** (HTML/CSS layout) with JS run by **duktape**. There is **no live DOM and no browser runtime**.

⚠️ **There is no mutable document.** None of these exist or do anything — do not reach for them:

| Web API you might reflexively use | Reality in MUPF |
|---|---|
| `document.getElementById`, `el.innerHTML = …`, `el.onclick = …` | no DOM object to mutate |
| `addEventListener` | no event system — clicks arrive via `__mupf_click(x, y)` (see [Client API](MUPF-Client-API.md#receiving-input--__mupf_clickx-y)) |
| `setTimeout` / `setInterval` | **not available** — no timers |
| `fetch` / `XMLHttpRequest` / `WebSocket` | **not available** — talk to your server only via `MUPF.invoke` |
| `localStorage`, `cookie`, `window.location` | not available |

The only way to change the screen is to build a complete HTML string and call `MUPF.render(html)`; the host re-parses and repaints. State lives in **JS globals**, which survive across renders because the duktape heap is created once and never rebuilt (per re-render):

```js
var rows = [], page = 0;          // persists across MUPF.render()

function view() {                 // build the WHOLE page as a string
  var h = '<style>' + CSS + '</style><div id="bd">';
  for (var i = 0; i < rows.length; i++) h += '<div>' + esc(rows[i].name) + '</div>';
  return h + '</div>';
}
function render() { MUPF.render(view()); }

MUPF.invoke('getData', {}, function(resp){ rows = resp.rows || []; render(); });
```

⚠️ **Re-rendered views must contain no `<script>`.** Inline scripts run once, when the *entry* `index.html` loads. Because the duktape heap is **not** recreated on `MUPF.render()`, a `<script>` in a re-rendered fragment would re-run and **reset your globals** (re-zeroing `page`, etc.). Keep all JS in `index.html`; every later `view()` is pure markup.

🔒 **Never write a literal `<script>` tag inside an HTML comment or visible text** — not even to document it. `LoadHtml` extracts inline JS with a text scan: it strips `<!-- … -->` first, then finds the first script-open / next script-close. A literal opening script tag sitting in a comment or string is mistaken for the real block, corrupts the extracted JS, the duktape parse fails, and your bootstrap `MUPF.invoke` never runs. **Symptom:** the window opens, shows the "Loading…" splash forever, and there is no request in the server log. If you must mention the tag in a comment, break it up (e.g. `scr` + `ipt`).

### 7.2 CSS: what does and doesn't render

litehtml lays out, but drawing is done with GDI + a small SVG rasterizer. A lot of modern CSS is silently ignored. **Supported:** solid `background-color`, solid `border` / `border-*`, 2-stop `linear-gradient`, text (color/size/weight/`letter-spacing`/`text-transform`/`text-shadow`), `flex`, tables, `nth-child`, box metrics, `white-space:nowrap`, `position:absolute/relative`, `overflow:hidden`.

⚠️ **Not rendered — your layout must not depend on these:**

| CSS feature | Result | Do this instead |
|---|---|---|
| `border-radius` | corners drawn square | square panels, or bake rounding into an SVG |
| `box-shadow` | nothing | fake depth with a border + `linear-gradient` |
| `radial-gradient` / `conic-gradient` | nothing | use a 2-stop `linear-gradient` |
| `transition` / `@keyframes` / animation | nothing | there is no per-frame animation channel |
| `:hover` and any mouse-move state | never triggers | no hover/move event reaches JS — design for clicks only |
| `%` heights vs. the viewport | unreliable | hardcode px (the window size is fixed — see 7.4) |

💡 The window is a fixed size, so you can lay out against exact pixels (e.g. a 554px content area under a 38px title bar). Hardcode it.

### 7.3 Fonts: missing glyphs render as boxes (tofu)

Text is drawn with `TextOutW` and **has no font fallback**. A glyph the chosen font lacks renders as an empty box.

⚠️ This bites with symbol characters. Triangle arrows `◀ ▶` (U+25C0 / U+25B6) are **not** in Segoe UI / Tahoma and show as boxes. `✕` (U+2715) and `★` *do* render. The reliable fix is to **draw the icon as SVG** rather than rely on a glyph:

```html
<img class="pg" src="prev.svg"><img class="pg" src="next.svg">
```

### 7.4 Images are SVG-only; the window size is fixed

`<img>` and CSS `background-image` go through the **nanosvg** rasterizer. Reference assets **relative** to `client/` (`src="crown.svg"`), and ship the `.svg` in `client/` (it is packed automatically).

⚠️ **SVG only — PNG/JPG are not supported yet.** nanosvg handles paths, `rect`/`circle`/`ellipse`/`line`/`polygon`, flat fills, and `linearGradient`. It does **not** render SVG `<text>` — overlay text in HTML on top of the SVG, never inside it.

⚠️ **The window does not resize.** Its size is `client.window.width`/`height` from `plugin.json` (and `client.launcher.width`/`height` for a launcher). Read those values into your JS `W`/`H` constants and hit-test against them in `__mupf_click`. The host reserves the **top ~38px as a drag strip** and a **top-right ~38×38 area as a built-in close hotspot** — draw your title bar there; clicks elsewhere arrive in `__mupf_click(x, y)`.

### 7.5 All client input is untrusted — validate on the server

The server only **routes** a request to a plugin by `pluginId`; it does **not** vouch for the request contents. A modified client can send any `fn` and any `args` to any plugin it knows the id of.

🔒 **`pluginId` routing is not authorization.** Treat `fn` and `args` in `OnInvoke` as hostile. Whitelist `fn`, then validate every field's type and range before use:

```lua
function Plugin.OnInvoke(ctx, fn, args, reqId)
    if fn ~= "getRanks" then
        ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
        return
    end
    -- args is UNTRUSTED: clamp, never trust client-supplied numbers/strings
    local page = tonumber(args and args.page) or 0
    if page < 0 then page = 0 elseif page > 9 then page = 9 end
    -- ... use the clamped value, never the raw arg
end
```

🔒 **Never build SQL by string concatenation.** Always use `ctx:sql(sql, params, cb)` with `?` placeholders — the host validates each numeric/bool and escapes each string. Concatenating an untrusted `args` value into the query string is an injection hole even though the bridge is parameterized.

💡 The arg/reply payloads are real JSON↔Lua, but nesting is **capped at depth 32** to bound stack use — keep payloads flat-ish. `ctx:reply`'s body must be a **table** (or `nil`); passing a non-table raises a Lua error. An empty Lua table serializes to `[]` (a JSON array), so use a keyed table when you need a JSON object. Call `ctx:reply` **exactly once** per `reqId`.

### 7.6 Rate limit, message size, and package caps

These are enforced by the host. Exceeding them does **not** crash or disconnect the plugin — the offending request is dropped (and answered with an error if a reply was expected).

| Limit | Value | Enforced where / effect |
|---|---|---|
| Request rate | **20 calls / second per (player, plugin)** | `PluginMgr::RateAllow` token bucket (1000ms window). Over-limit ⇒ reply `{"error":"rate_limited","code":"RATE"}`; the socket is never closed. |
| Message payload | **`maxMsgBytes` 8000 bytes** (`MAX_MSG_BYTES`) | per envelope (your `args` JSON and each `ctx:reply` JSON). Oversize ⇒ the envelope fails to encode and is dropped. |
| `fn` / channel name | **64 bytes** (`MAX_NAME_LEN`) | longer names are rejected by the envelope. |
| In-flight requests | **`maxPending`** (manifest, e.g. 16) | the client caps concurrent un-answered `MUPF.invoke` calls. |
| Per-asset / per-entry | **`maxAssetBytes` 64 KB** (manifest); host hard cap `PACK_MAX_ENTRY_BYTES` 4 MB | a single client file. |
| Whole `.mupf` (uncompressed) | **`PACK_MAX_TOTAL_BYTES` 4 MB** total, **`PACK_MAX_ENTRIES` 256** files, path ≤ 255 | enforced while unpacking (anti zip-bomb); an oversize/over-count archive is rejected and the plugin is quarantined. |

⚠️ **Don't try to stream a 100KB result in one reply.** With an 8000-byte cap you must **paginate** server-side (the Ranking plugin returns at most 100 rows in pages) — don't dump a whole table in a single `ctx:reply`.

💡 The ~20/s limit is generous for click-driven UIs but unforgiving for accidental loops. Because there are no timers, the only way to exceed it is to fire `MUPF.invoke` from many clicks or from a render that re-invokes — guard against re-entrant fetches.

### 7.7 Reload semantics — when your edits actually take effect

This trips up everyone during development. The server and client halves refresh on **different** triggers.

| You edited… | Takes effect when |
|---|---|
| `server/server.lua` (logic, SQL, manifest caps) | **GS restart**, or the GameServer-window **`Reload Script`** command — it re-scans + re-packs every plugin and reloads each `server.lua` into the Lua stack **without** restarting the GS. `server.lua` is loaded **once** into the Lua state; a plain file save alone does nothing. |
| `client/*` (index.html, CSS, SVG, launcher.html) | the player must **relog**. Reconnecting re-streams the fresh client files. Players already online keep the **old** UI until they reconnect; new logins get the new version immediately. In dev (`MUPF_DEV_HOTRELOAD = 1`) the connect path re-packs from disk, so the loop is just *edit client file → relog → see it*. |

⚠️ **`MUPF_DEV_HOTRELOAD` (in `Game/PluginMgr.cpp`) MUST be `0` in production.** It defaults to `1` for the edit-relog loop; left on, the server **re-zips every plugin on every single login**. In production set it to `0` and ship `.mupf` files — the packed client blob is cached and streamed without per-login re-packing.

💡 If a `Reload Script` doesn't seem to change anything, you almost certainly edited a **client** file and need a relog — `Reload Script` refreshes the server side immediately but the connected client keeps its already-streamed UI.

### 7.8 Pre-ship checklist

Run through this before packing and dropping a `.mupf` into production:

- [ ] **No browser APIs** — searched the client JS for `document.`, `addEventListener`, `setTimeout`, `setInterval`, `fetch`, `XMLHttpRequest`; none present.
- [ ] **Single `<script>`, only in `index.html`** — every `MUPF.render()` view is pure markup; no fragment carries a `<script>`.
- [ ] **No literal script tag in any comment/text** anywhere in the HTML (it breaks the JS extractor → permanent "Loading…").
- [ ] **State in globals** — paging/selection state is in JS globals that survive `MUPF.render()`, not regenerated on each render.
- [ ] **CSS sanity** — layout still reads correctly with `border-radius`, `box-shadow`, `radial-gradient`, `:hover`, transitions all treated as no-ops; no `%` heights relied on.
- [ ] **No tofu** — every symbol is a font glyph that renders (`✕`, `★`) or an **SVG** icon (arrows, badges, frames). No `◀`/`▶`.
- [ ] **SVG only** — all images are `.svg` (no PNG/JPG); SVG text is overlaid in HTML, not inside the SVG.
- [ ] **Fixed-size layout** — `W`/`H` in JS match `client.window` (and launcher pages match `client.launcher`); `__mupf_click` hit-tests those pixels; title bar drawn under the top ~38px, close at top-right.
- [ ] **Server validates every request** — `fn` is whitelisted; every `args` field is type/range-checked and clamped; `pluginId` is never treated as authorization.
- [ ] **All SQL is parameterized** — `ctx:sql` with `?` placeholders only; no string-concatenated queries.
- [ ] **`ctx:reply` once per `reqId`**, body is a table, payload stays under 8000 bytes (paginate large result sets); nesting under depth 32.
- [ ] **Capabilities minimal** — `plugin.json permissions` lists only what's used, and `policy.json` grants exactly that intersection.
- [ ] **Within size caps** — each client file ≤ `maxAssetBytes` (64 KB) and the whole plugin ≤ 4 MB / 256 files.
- [ ] **Manifest `id` matches `PluginRegister(...)`** in `server.lua` exactly.
- [ ] **Packed with MupfPacker** into a `.mupf` and dropped in `Data/ClientLuaPlugin/`; `MUPF_DEV_HOTRELOAD = 0`; tested by a **fresh relog** (so the client re-streams the shipped files).
