# DevEmu Lua and MUPF Developer Documentation

Public documentation for developers who build server scripts and server-delivered client UI plugins for DevEmu MU Online.

## Choose the correct development model

| You want to build | Start here |
|---|---|
| A normal GameServer Lua script that uses the full engine API | [Getting Started](docs/Getting-Started.md) |
| A server-managed plugin with an in-game HTML/CSS/JavaScript UI | [MUPF System Overview](docs/MUPF-Client-System-Overview.md) |

MUPF is a separate plugin contract. It does not replace ordinary GameServer Lua scripts, and it is not the legacy `Data/LuaUI` delivery path.

## Server-side Lua reference

- **[Server Lua Functions](docs/Server-Lua-Functions.md)** — code-generated reference for all exported engine functions, including exact argument counts.
- [Server Callbacks](docs/Server-Callbacks.md) — event hooks, parameters, and return values.
- [Server Global Functions](docs/Server-Global-Functions.md) — narrative API reference grouped by topic.
- [Player Structure](docs/Player-Structure.md) — player getters and setters.
- [Monster Structure](docs/Monster-Structure.md) — monster access, spawn, and iteration.
- [Item Structures](docs/Item-Structures.md) — inventory, item delivery, and Gremory Case.
- [Database Structures](docs/Database-Structures.md) — asynchronous SQL patterns.
- [Scheduler](docs/Scheduler.md) — timers, daily jobs, cooldowns, and periodic work.
- [Project Layout](docs/Project-Layout.md) — script folders, `ScriptMain`, and require paths.

## MU Plugin Framework (MUPF)

MUPF combines private GameServer Lua with a client UI made from HTML, CSS, JavaScript, SVG, and supported raster images. Approved client assets are delivered by the GameServer when the player connects; plugin developers do not ship native client DLLs.

Read these in order:

1. **[MUPF System Overview](docs/MUPF-Client-System-Overview.md)** — architecture, lifecycle, scope, and current implementation status.
2. **[Writing Plugins](docs/Writing-Plugins.md)** — practical tutorial and reusable patterns.
3. [MUPF Plugins](docs/MUPF-Plugins.md) — authoritative manifest, permissions, deployment, limits, and packaging contract.
4. [MUPF Server API](docs/MUPF-Server-API.md) — `PluginRegister`, `OnInvoke`, `host`, and `ctx:*`.
5. [MUPF Client API](docs/MUPF-Client-API.md) — `MUPF.invoke`, rendering, input, windows, CSS, and images.

> Documentation status: audited against the current `dev` implementation on **2026-08-31**. Where a planned API exists in code but is not complete end-to-end, the documentation marks it as provisional instead of presenting it as supported.
