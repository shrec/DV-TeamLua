# DevEmu — Lua Scripting API

Documentation for server-side and client-side Lua plugin development for the DevEmu MU Online server.

---

## Getting Started

- [Getting Started](docs/Getting-Started.md) — first plugin, constants, common patterns
- [Project Layout](docs/Project-Layout.md) — folder structure, ScriptMain, require paths

---

## Server-Side Lua

- **[Server Lua Functions](docs/Server-Lua-Functions.md)** — the complete, code-generated reference of **all 244 engine functions** (`LuaFunction.cpp`): exact **argument counts**, parameter types, and returns. Authoritative — arg counts are enforced, so match them exactly (the #1 cause of plugin crashes).
- [Server Callbacks](docs/Server-Callbacks.md) — all 29 event hooks with parameters and return values
- [Server Global Functions](docs/Server-Global-Functions.md) — narrative C++ API reference (grouped by topic with examples)
- [Player Structure](docs/Player-Structure.md) — all player getters and setters
- [Monster Structure](docs/Monster-Structure.md) — spawn, kill events, map iteration
- [Item Structures](docs/Item-Structures.md) — inventory, give/drop, Gremory Case
- [Database Structures](docs/Database-Structures.md) — async SQL guide with examples
- [Scheduler](docs/Scheduler.md) — timer, daily reset, cooldowns, periodic DB flush

---

## MU Plugin Framework (MUPF)

Build a self-contained plugin with **both** a server side (Lua) and an in-game client UI
(HTML/CSS/JS/SVG), delivered to players at runtime.

- **[Writing Plugins](docs/Writing-Plugins.md)** — the complete how-to guide: build a plugin from hello-world to any complexity (**start here**)
- [MUPF Plugins](docs/MUPF-Plugins.md) — overview, architecture, the manifest, capabilities, dev-vs-ship workflow, packaging (`.mupf` + MupfPacker)
- [MUPF Server API](docs/MUPF-Server-API.md) — `PluginRegister`, `OnInvoke`, and the `ctx:*` object (`reply` / `sql` / `playerName` / `push`)
- [MUPF Client API](docs/MUPF-Client-API.md) — the in-game UI: `MUPF.invoke` / `render` / `close`, the no-DOM re-render model, `__mupf_click`, SVG, CSS/font caveats
