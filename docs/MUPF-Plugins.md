# MUPF Plugins

**MUPF** (MU Plugin Framework) lets you ship a self-contained plugin that has **both**
a server side and a client side:

- **Server Lua** (`server/server.lua`) — runs inside the GameServer, can read player info
  and query the database. **Never leaves the server.**
- **Client UI** (`client/`) — HTML/CSS/JS (+ optional SVG) rendered in-game in its own
  window, delivered to the player at runtime.

A plugin talks to its own server logic over a request/response bridge — the client calls a
server function, the server replies, the client re-renders. The bundled **Server Ranking**
plugin (`com.dvteam.ranking`) is a complete working example.

---

## Architecture at a glance

```
        PLUGIN (one folder, or one .mupf file)
        ├─ plugin.json        ← manifest (id, window, hotkey, permissions)
        ├─ server/server.lua  ← runs on the GameServer (DB + player access)
        └─ client/            ← delivered to the player
           ├─ index.html      ← UI (HTML/CSS/JS, the entry page)
           ├─ *.svg           ← vector icons (optional)
           └─ client.lua      ← optional client-side Lua

   GameServer  ──BEGIN/PKG/READY──►  Client (in-game window)
        ▲   server.lua loaded            │   index.html rendered
        │   here, in the Lua stack       │
        └──────── request / reply ───────┘
           MUPF.invoke(fn,args,cb)  ⇄  OnInvoke(ctx,fn,args,reqId) → ctx:reply
```

On connect the server streams each plugin's **client** files to the player (server Lua is
never sent). The client unpacks them into a per-plugin sandbox and renders `index.html`.

---

## Plugin layout

```
com.dvteam.ranking/
├─ plugin.json
├─ server/
│  └─ server.lua          -- PluginRegister(...) + OnInvoke + ctx:sql
└─ client/
   ├─ index.html          -- the UI page (entry)
   ├─ client.lua          -- optional
   ├─ crown.svg           -- assets
   └─ ...
```

A plugin lives under the server's plugins folder: **`Data/ClientLuaPlugin/`**.

---

## The manifest — `plugin.json`

```json
{
  "manifest": 1,
  "id": "com.dvteam.ranking",
  "name": "Server Ranking",
  "version": "1.0.0",
  "apiVersion": 1,
  "minApiVersion": 1,
  "author": "DV Team",
  "description": "Top-100 character ranking panel.",

  "server": { "entry": "server/server.lua" },

  "client": {
    "entry": "client/client.lua",
    "html":  "client/index.html",
    "assets": ["client/index.html", "client/crown.svg"],
    "window": { "width": 640, "height": 560, "resizable": false, "title": "Server Ranking" }
  },

  "entryPoints": {
    "hotkey":   "F5",
    "uiButton": { "icon": null, "label": "Ranking", "tooltip": "Show the server ranking" }
  },

  "permissions": ["ui.window", "ui.hotkey", "net.request", "db.query"],

  "limits": { "maxAssetBytes": 65536, "maxMsgBytes": 8000, "maxPending": 16 }
}
```

| Field | Meaning |
|---|---|
| `id` | Reverse-DNS unique id. The routing + policy key. **Required.** |
| `apiVersion` / `minApiVersion` | Host-API version the plugin targets / requires. A client whose API < `minApiVersion` quarantines the plugin. |
| `server.entry` | Path to the server Lua (loaded server-side, never sent). |
| `client.html` | the HTML entry page — the UI **and its JavaScript**. |
| `client.entry` | *reserved.* A per-plugin **client-side Lua** runtime is not active yet — **client logic is JavaScript** in your HTML, not Lua. Safe to omit. |
| `client.window` | `width`, `height`, `title` of the in-game main window. |
| `client.launcher` | *(optional)* a tier-1 always-on launcher button — see [2-tier plugins](#2-tier-plugins--an-always-on-launcher-button). |
| `entryPoints.hotkey` | Key that toggles the window (`"F5"`, `"F6"`, a single letter…). |
| `permissions` | Capabilities the plugin requests. Granted only if the operator also allows them (see Capabilities). |
| `limits` | Per-plugin caps (asset bytes, message bytes, in-flight requests). |

---

## Capabilities (permissions)

A plugin only gets a capability if it is in **both** the manifest `permissions` **and** the
operator's `Data/ClientLuaPlugin/policy.json` (intersection — **deny-by-default**). An unknown
plugin gets nothing.

```json
// Data/ClientLuaPlugin/policy.json
{
  "com.dvteam.ranking": ["ui.window", "ui.hotkey", "net.request", "db.query"]
}
```

| Capability | Grants |
|---|---|
| `db.query` | `ctx:sql(...)` — parameterized async SQL |
| `player.readBasic` | `ctx:playerName()`, `ctx:playerLevel()` |
| `net.serverPush` | `ctx:push(...)` — server→client push |
| `ui.window` | `ctx:open()` and the plugin window |
| `ui.hotkey` | the launcher hotkey |
| `net.request` | the client may call the server (`MUPF.invoke`) |

> ⚠️ A `ctx:*` call without its capability raises a Lua error (caught + logged) — the call
> fails, the plugin keeps running.

See **[MUPF Server API](MUPF-Server-API.md)** and **[MUPF Client API](MUPF-Client-API.md)**.

---

## 2-tier plugins — an always-on launcher button

A hotkey is not always enough — you can't bind every plugin to a key. A plugin can ship a
**tier-1 launcher**: a small, always-on, fixed button that opens the plugin's **tier-2 main
window** on click. Both are ordinary HTML pages (same engine, same rules) — it's just two
windows of one plugin.

Declare the launcher under `client.launcher`:

```json
"client": {
  "html": "client/index.html",
  "launcher": {
    "html": "client/launcher.html",
    "width": 150, "height": 46,
    "anchor": "top-left", "x": 16, "y": 96
  },
  "window": { "html": "client/index.html", "width": 420, "height": 300, "title": "2-Tier Test" }
}
```

| Field | Meaning |
|---|---|
| `html` | the launcher button's HTML page (e.g. `client/launcher.html`) |
| `width` / `height` | the launcher window size (px) |
| `anchor` | where on the game window it sits (below) |
| `x` / `y` | pixel offset from that anchor, into the game client area |

**Anchors:** `top-left` · `top-right` · `top-center` · `bottom-left` · `bottom-right` ·
`bottom-center` · `left` · `right` · `center`. The launcher is positioned **relative to the game
window** (works at any resolution), follows it when it moves, and hides when you alt-tab away.

The launcher window is **always visible, fixed, not movable, and not closable** — it is the
plugin's entry point. Its page opens the main window by calling **`MUPF.open()`** (see the
[Client API](MUPF-Client-API.md#mupfopenid--tier-1-launcher)). The main window opens / closes /
drags normally; a plugin may also keep an `entryPoints.hotkey` and/or be opened by the server.

```
[always-on launcher button]  --click → MUPF.open()-->  [main window opens]
   client/launcher.html                                   client/index.html
```

The bundled **2-Tier Test** plugin (`com.dvteam.test2tier`) is a minimal working example
(launcher pill → a main window that pings the server).

---

## Developing a plugin (dev vs ship)

You do **not** re-pack on every edit. There are two modes:

### Dev mode — work in a source folder

Put the plugin **folder** under `Data/ClientLuaPlugin/` and run the server in dev mode
(`MUPF_DEV_HOTRELOAD = 1`, the default in dev builds). The server re-reads the plugin from
disk on each connect, so:

```
edit client/index.html  →  relog  →  see the change
```

No packing, no full server restart. (Editing `server.lua` still needs a server restart — it
is loaded once into the Lua stack at startup.)

### Ship — pack into one .mupf

When the plugin is ready, pack it into a single encrypted file with **MupfPacker**:

```bat
MupfPacker.exe Data\ClientLuaPlugin\com.dvteam.ranking
:: -> com.dvteam.ranking.mupf
```

Drop the `.mupf` into the server's `Data/ClientLuaPlugin/` folder. In production
(`MUPF_DEV_HOTRELOAD = 0`) the server **enumerates `*.mupf` files**, loads each plugin's
`server.lua` into the Lua stack, and streams the client files to players.

> The GameServer reads **both** entry types from the same folder: **folders** (dev) and
> **`.mupf` files** (shipped). You can mix them.

### Reloading at runtime (no GS restart)

The GameServer-window **`Reload Script`** command also reloads plugins: it re-scans + re-packs
every plugin from disk (folders + `*.mupf`) and reloads each plugin's `server.lua` into the Lua
stack — **without restarting the GameServer**. Use it after editing a plugin (or dropping in a
new `.mupf`).

- **Server side** (`server.lua`, manifests, packed client blobs) refreshes immediately.
- **Client side**: players already online keep the old UI until they **relog** (reconnecting
  re-streams the fresh client files). New logins get the new version right away.

---

## The `.mupf` package

A `.mupf` is the single distributable file for a plugin:

```
"MUPF" + version + XOR(secret, ZIP{ plugin.json, server/…, client/… })
```

- A ZIP of the **whole** plugin, then XOR-encrypted with an embedded secret.
- **Self-describing:** the server/client accept either this encrypted form or a plain `PK…`
  zip (dev), so the same code path handles both.
- The cipher + key live in exactly one place (`Common/MupfPack.cpp`), shared by the server,
  the client, and **MupfPacker** — there is no second copy of the key.

> 🔒 Build **MupfPacker** as a C++ exe and protect it with **Themida** (like the client DLL).
> The secret is compiled in; a plaintext/script packer would leak it and let anyone forge or
> decrypt packages. See `tools/MupfPacker/README.md`.

The server unpacks a `.mupf` in memory, loads `server.lua` **directly into the Lua stack**,
and re-packs only the **client** files (manifest + `client/*`, never `server/`) to send to
players.

---

## Operations checklist

| Setting | Dev | Production |
|---|---|---|
| `MUPF_DEV_HOTRELOAD` (`Game/PluginMgr.cpp`) | `1` (relog picks up edits) | **`0`** (cached, no per-login re-pack) |
| Plugin form | source folder | `.mupf` file |
| Wire blob | plain zip | encrypted |
| `policy.json` | grant caps you test | grant only what each plugin needs |

> ⚠️ `server.lua` is loaded **once at startup**. Adding/removing a plugin or editing its
> server Lua requires a server restart (the client side hot-reloads on relog in dev).

---

## See also

- **[MUPF Server API](MUPF-Server-API.md)** — `PluginRegister`, `OnInvoke`, the `ctx:*` object.
- **[MUPF Client API](MUPF-Client-API.md)** — the in-game UI: `MUPF.invoke/render/close`, clicks, SVG.
- **[Database Structures](Database-Structures.md)** — the async SQL model `ctx:sql` builds on.
