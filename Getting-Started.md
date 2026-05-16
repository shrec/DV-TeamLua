# Getting Started

This guide walks you through writing your first server-side Lua plugin for DevEmu.

---

## How It Works

The game server maintains a single Lua state. Communication is two-directional:

| Direction | Mechanism |
|---|---|
| **Engine → Lua** | Engine calls `BridgeFunction_*` callbacks you register |
| **Lua → Engine** | You call global C++ functions like `GetObjectName()`, `NoticeSend()` |

Every player and monster is referenced by an **integer index** (`aIndex`). Pass this index to every API call — there are no object pointers in Lua.

---

## File Layout

```
LuaBridge/
├── ScriptMain.lua           ← engine entry point (require your plugin here)
├── System/
│   ├── ScriptCore.lua       ← BridgeFunctionAttach() and callback dispatch
│   ├── ScriptDefine.lua     ← constants: classes, maps, object states
│   └── ScriptReader.lua     ← flat-file parser helper
└── MyPlugin/
    ├── MyPlugin.lua         ← your plugin logic
    └── Config.lua           ← your settings
```

---

## Minimal Plugin

```lua
-- LuaBridge/MyPlugin/MyPlugin.lua

local Switch = true   -- set false to disable

BridgeFunctionAttach("OnReadScript", function()
    LogPrint("[MyPlugin] loaded")
end)

BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    if not Switch then return end
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end

    local name = GetObjectName(aIndex)
    NoticeSend(aIndex, 0, "Welcome, " .. name .. "!")
end)
```

Register it in `ScriptMain.lua`:

```lua
-- LuaBridge/ScriptMain.lua
require("MyPlugin/MyPlugin")
```

---

## Essential Constants

Loaded automatically from `System/ScriptDefine.lua`:

```lua
-- Connection states
OBJECT_OFFLINE   = 0   -- socket exists, not logged in
OBJECT_CONNECTED = 1   -- connected to server
OBJECT_LOGGED    = 2   -- account authenticated
OBJECT_ONLINE    = 3   -- character in world

-- Object types
OBJECT_NONE    = 0
OBJECT_USER    = 1
OBJECT_MONSTER = 2
OBJECT_NPC     = 3
OBJECT_ITEM    = 4

-- Character classes
CLASS_DW = 0    CLASS_DK = 1    CLASS_FE = 2
CLASS_MG = 3    CLASS_DL = 4    CLASS_SU = 5
CLASS_RF = 6    CLASS_GL = 7    CLASS_RW = 8
CLASS_SL = 9    CLASS_GC = 10   CLASS_KM = 11
CLASS_LM = 12   CLASS_IK = 13
```

---

## Common Patterns

### Guard before accessing player data
```lua
-- Always check state before reading player properties
if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
if GetObjectType(aIndex) ~= OBJECT_USER then return end
```

### Throttle the timer (fires every second)
```lua
local tick = 0
BridgeFunctionAttach("OnTimerThread", function()
    tick = tick + 1
    if tick % 60 ~= 0 then return end  -- once per minute
    -- your work here
end)
```

### Async database query
```lua
BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    local name = GetObjectName(aIndex):gsub("'", "''")
    SQLAsyncQuery("myplugin_load", string.format(
        "SELECT coins FROM myplugin_data WHERE char_name = '%s'", name))
end)

BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if label ~= "myplugin_load" then return end
    -- rows[1] is the first row (nil if no results)
    local coins = tonumber(rows[1] and rows[1]["coins"] or 0)
end)
```

### Command handler
```lua
BridgeFunctionAttach("OnCommandManager", function(aIndex, command)
    if command == "/mycommand" then
        local arg = CommandGetArgString(1)  -- first argument after command
        NoticeSend(aIndex, 0, "You typed: " .. tostring(arg))
        return 1  -- 1 = command handled
    end
    return 0
end)
```

---

## Next Steps

| Page | What you'll find |
|---|---|
| [Server Callbacks](Server-Callbacks) | Every event hook with parameters and return values |
| [Server Global Functions](Server-Global-Functions) | Complete C++ API reference |
| [Player Structure](Player-Structure) | All player getters and setters |
| [Item Structures](Item-Structures) | Inventory and item API |
| [Database Structures](Database-Structures) | Async SQL guide |
| [Scheduler](Scheduler) | Timer and cron-style scheduling |
