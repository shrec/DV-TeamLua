# MUPF Server API

The server side of a plugin is `server/server.lua`. It runs inside the GameServer's Lua state
in a **fresh per-plugin environment** (writes stay local; reads fall through to the trusted
server `_G`). It registers a hooks table and handles requests from its client UI.

> All `ctx:*` calls are **capability-gated** (see [MUPF Plugins → Capabilities](MUPF-Plugins.md#capabilities-permissions)).
> A call without its capability raises a Lua error (caught + logged); the plugin keeps running.

---

## Skeleton

```lua
local Plugin = {}

function Plugin.OnLoad()
    host.log("my plugin ready")          -- optional; runs once at load
end

-- A client request arrived. fn/args are UNTRUSTED — validate before use.
function Plugin.OnInvoke(ctx, fn, args, reqId)
    if fn == "ping" then
        ctx:reply(reqId, { pong = true, you = ctx:playerName() })
        return
    end
    ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
end

PluginRegister("com.yourteam.yourplugin", Plugin)   -- id MUST match plugin.json
return Plugin
```

---

## `PluginRegister(idStr, hooks)`

Registers the plugin's hooks. Call it once at the end of `server.lua`. `idStr` must equal the
manifest `id`.

| Hook | When |
|---|---|
| `OnLoad()` | once, right after the plugin loads (optional) |
| `OnInvoke(ctx, fn, args, reqId)` | a client `MUPF.invoke(fn, args, cb)` arrived |

> ⚠️ A plugin that never calls `PluginRegister` (or provides no `OnInvoke`) receives no
> invocations — the client gets a `no_handler` error.

---

## `host.log(...)`

Writes to the server log (`plugin` channel). Available without a capability.

```lua
host.log("loaded", someValue)
```

---

## The `ctx` object

`ctx` is passed to `OnInvoke`. It is scoped to the **calling player** and the **current
request**.

### `ctx:aIndex()` → number

The caller's in-world object index (the player slot). Always available.

### `ctx:playerName()` → string  · `ctx:playerLevel()` → number
*Capability: `player.readBasic`.* Basic info about the calling player. Return `nil`/empty if
the player logged out mid-request.

```lua
local name = ctx:playerName()
local lvl  = ctx:playerLevel()
```

### `ctx:reply(reqId, table)`
Sends the response for this request. `table` is converted to JSON and delivered to the
client's `MUPF.invoke` callback. Call it **once** per `reqId`.

```lua
ctx:reply(reqId, { rows = out, total = #out })
-- error replies are a normal table; pick your own shape:
ctx:reply(reqId, { error = "bad_input", code = "BAD_REQUEST" })
```

### `ctx:sql(sql, params, cb)`
*Capability: `db.query`.* Runs a **parameterized, asynchronous** query. The game loop is never
blocked; `cb(rows)` fires when the result arrives. Each `?` in `sql` is replaced by the
matching `params` entry — **the host escapes/validates each value**, so never build SQL by
string concatenation.

```lua
ctx:sql(
    "SELECT name, reset, level FROM character_info WHERE authority = ? ORDER BY reset DESC LIMIT 100",
    { 0 },                                  -- params: ? -> 0  (numbers validated, strings escaped, bools -> 0/1)
    function(rows)
        local out = {}
        if rows then
            for i, r in ipairs(rows) do      -- rows = 1-based array of row tables
                out[i] = { rank = i, name = r.name, reset = tonumber(r.reset) or 0 }
            end
        end
        ctx:reply(reqId, { rows = out })     -- reply from inside the SQL callback
    end)
```

`rows` follows the same shape as the global async-SQL model — see
**[Database Structures](Database-Structures.md)** (`SELECT` with results → 1-based table of
`{ col = value }`; otherwise a number).

> 🔒 The callback is routed by a **host-allocated opaque label** — one plugin can never
> receive another plugin's SQL result.

### `ctx:push(channel, table)` · `ctx:open()`
*Capabilities `net.serverPush` / `ui.window`.* Server-initiated push to the client / request to
open the plugin window.

> 🚧 The server side of `ctx:push` / `ctx:open` exists; client-side delivery (push → client
> handler, server-initiated open) is being finalized. Use request/reply (`OnInvoke` +
> `ctx:reply`) as the primary pattern for now.

---

## JSON ↔ Lua

- `args` in `OnInvoke` is the client's payload, already parsed into a **Lua table**
  (objects → tables, arrays → 1-based tables, numbers/strings/bools/null → Lua values).
- The table you pass to `ctx:reply` / `ctx:push` is serialized back to JSON for the client.
- Nesting is capped (depth 32) to bound stack use. Keep payloads flat-ish.

> ⚠️ **Treat `fn` and `args` as hostile.** A modified client can send any `fn`/`args` to any
> plugin. Validate types and ranges before use (the Ranking plugin clamps its `page` arg, for
> example). The server only *routes* by plugin id; it does not vouch for the request contents.

---

## Full example — Server Ranking (`server/server.lua`)

```lua
local Ranking = {}
local PAGE_SIZE, MAX_ROWS = 10, 100

local RANK_SQL =
    "SELECT guid, name, race, reset, level, level_master, level_majestic " ..
    "FROM character_info WHERE authority = 0 " ..
    "ORDER BY `reset` DESC, `level_majestic` DESC, `level_master` DESC, `level` DESC LIMIT 100"

function Ranking.OnLoad()
    host.log("ranking server ready")
end

function Ranking.OnInvoke(ctx, fn, args, reqId)
    if fn ~= "getRanks" then
        ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
        return
    end
    ctx:sql(RANK_SQL, {}, function(rows)
        local out = {}
        if rows then
            for i, r in ipairs(rows) do
                if i > MAX_ROWS then break end
                out[#out + 1] = {
                    rank = i, name = r.name or "", class = tonumber(r.race) or 0,
                    reset = tonumber(r.reset) or 0, level = tonumber(r.level) or 0,
                    ml = tonumber(r.level_master) or 0, jl = tonumber(r.level_majestic) or 0,
                }
            end
        end
        ctx:reply(reqId, { total = #out, rows = out })
    end)
end

PluginRegister("com.dvteam.ranking", Ranking)
return Ranking
```

The matching client (`index.html`) calls `MUPF.invoke("getRanks", {}, cb)` and renders the
result — see **[MUPF Client API](MUPF-Client-API.md)**.
