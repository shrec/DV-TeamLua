# MUPF Server API

The server side of a plugin is `server/server.lua`. It runs in the GameServer Lua state inside a per-plugin environment: plugin writes remain local, while reads can fall through to the trusted server globals.

`server.lua` is private server code. It is not included in the client package sent to players.

## Minimal server plugin

```lua
local Plugin = {}

function Plugin.OnLoad()
    host.log("plugin ready")
end

function Plugin.OnInvoke(ctx, fn, args, reqId)
    if fn == "ping" then
        ctx:reply(reqId, { pong = true })
        return
    end

    ctx:reply(reqId, { error = "unknown_fn", code = "BAD_REQUEST" })
end

PluginRegister("com.example.plugin", Plugin)
return Plugin
```

The registration id must exactly match `plugin.json.id`.

## `PluginRegister(id, hooks)`

Registers the hook table for the plugin currently being loaded. A plugin cannot register hooks under another plugin's id.

| Hook | Called when |
|---|---|
| `OnLoad()` | Immediately after the server chunk loads. Optional. |
| `OnInvoke(ctx, fn, args, reqId)` | A client `MUPF.invoke(fn, args, callback)` request arrives. |

A missing registration or missing `OnInvoke` handler makes the server side inert and returns a handler error to request/reply calls.

## `host`

### `host.log(...)`

Writes values to the GameServer `plugin` log channel. No capability is required.

```lua
host.log("loaded", value)
```

### `host.open(aIndex, plugin)` / `host.openPlugin(aIndex, plugin)`

Accepts a numeric plugin id or string plugin id and requires the target plugin's `ui.window` capability. The server sender exists, but the current client host only logs the resulting server-open message and does not open the popup. Treat both names as provisional and do not use them in production workflows yet.

## The `ctx` request object

`ctx` belongs to the calling player and current plugin request.

### `ctx:aIndex()` → number

Returns the caller's in-world object index. No capability is required.

### `ctx:playerName()` → string or nil

*Capability: `player.readBasic`.*

Returns the caller's backend character name. Returns `nil` when the player is no longer available.

### `ctx:playerLevel()` → number

*Capability: `player.readBasic`.*

Returns the caller's current level, or `0` when the player is no longer available.

### `ctx:reply(reqId, table)`

Sends the correlated response to the JavaScript callback registered by `MUPF.invoke`.

```lua
ctx:reply(reqId, { rows = rows, total = #rows })
```

- Call it once for each request that expects a reply.
- The body must be a Lua table or `nil`/omitted.
- The serialized JSON payload must stay under 8,000 bytes.
- Application errors use your own table shape, for example `{ error = "bad_input", code = "BAD_REQUEST" }`.

### `ctx:sql(query, params, callback)`

*Capability: `db.query`.*

Runs a query asynchronously and invokes the callback when the result is returned to the GameServer Lua runtime.

```lua
ctx:sql(
    "SELECT name, level FROM character_info WHERE authority = ? ORDER BY level DESC LIMIT 100",
    { 0 },
    function(rows)
        local result = {}
        if rows then
            for i, row in ipairs(rows) do
                result[i] = {
                    name = row.name or "",
                    level = tonumber(row.level) or 0
                }
            end
        end
        ctx:reply(reqId, { rows = result })
    end)
```

Parameter rules:

- each `?` is replaced in order;
- numbers must be finite;
- strings are escaped and quoted by the host;
- booleans become `1` or `0`;
- other parameter types are rejected.

The current binding replaces `?` lexically, not through a native database prepared-statement object. Do not place literal `?` characters in quoted SQL text or comments, and never concatenate client input into SQL.

The callback is registered under a host-created opaque label, so one plugin cannot claim another plugin's SQL callback.

### `ctx:push(channel, table)` — provisional

*Capability: `net.serverPush`.*

The GameServer can encode and send the push envelope, but the current client host does not deliver it to a JavaScript callback. Use request/reply polling driven by explicit player actions for now.

### `ctx:open()` — provisional

*Capability: `ui.window`.*

The GameServer can encode and send the open envelope, but the current client host does not execute the open action. Use a manifest hotkey, `client/launcher.html`, or client-side `MUPF.open()`.

## JSON and Lua conversion

Inbound JSON is converted as follows:

- object → Lua table with string keys;
- array → 1-based Lua array table;
- string/number/boolean → matching Lua primitive;
- JSON `null` → `nil`.

Outbound Lua tables are arrays only when every key is an integer from `1` through `n`. An empty Lua table serializes as `[]`; use at least one string key when you need a JSON object.

Conversion is capped at 32 nested levels. Deeper values become `null`.

## Security requirements

The client payload is untrusted. A modified client may send arbitrary function names and arguments.

Every `OnInvoke` implementation must:

1. whitelist `fn`;
2. verify every argument type;
3. clamp numeric ranges and string lengths;
4. re-check ownership, currency, inventory space, and permissions on the server;
5. use `ctx:sql` parameters for any client-derived value;
6. keep replies small and paginated.

`pluginId` selects the destination plugin. It is not proof that the caller is authorized to perform the requested action.

## Reload behavior

`server.lua` is loaded when the GameServer Lua state is built. A file save alone does not update the live state.

After changing server Lua, the manifest, policy, or plugin list, use the GameServer's **Reload Script** command or restart the GameServer. Client UI changes still require the player to reconnect because online clients keep their previously streamed package.

## See also

- [MUPF Plugins](MUPF-Plugins.md) — manifest, policy, limits, and deployment.
- [MUPF Client API](MUPF-Client-API.md) — request callbacks and UI rendering.
- [Database Structures](Database-Structures.md) — general asynchronous SQL behavior.
