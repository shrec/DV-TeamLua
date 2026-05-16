# Project Layout

How the Lua bridge is organized on disk and how scripts are loaded.

---

## Directory Structure

```
LuaBridge/
├── ScriptMain.lua                  ← engine entry point
│
├── System/
│   ├── ScriptCore.lua              ← BridgeFunctionAttach + callback dispatch
│   ├── ScriptDefine.lua            ← constants (classes, maps, object types)
│   └── ScriptReader.lua            ← flat-file parser utility
│
├── MyPlugin/                       ← one folder per plugin
│   ├── MyPlugin.lua                ← plugin logic
│   └── Config.lua                  ← plugin settings
│
├── DailyReward/
│   ├── DailyReward.lua
│   └── Configs/
│       └── Configs.lua
│
└── WelcomeMessage/
    ├── Script.lua
    └── Config.lua
```

---

## Entry Point — `ScriptMain.lua`

The engine calls this file first. Use it only to `require` your plugins:

```lua
-- LuaBridge/ScriptMain.lua
require("System/ScriptCore")     -- loads callback system (required)
require("DailyReward/DailyReward")
require("WelcomeMessage/Script")
require("MyPlugin/MyPlugin")
```

Disable a plugin without deleting it — just comment it out:

```lua
-- require("MyPlugin/MyPlugin")   -- temporarily disabled
```

---

## `System/ScriptCore.lua`

Implements `BridgeFunctionAttach()`. Every callback goes through this file.

You should **not** edit it directly. Just call `BridgeFunctionAttach()` from your plugin files.

---

## `System/ScriptDefine.lua`

Constants auto-loaded before your code runs. Examples:

```lua
-- Connection states
OBJECT_OFFLINE   = 0
OBJECT_CONNECTED = 1
OBJECT_LOGGED    = 2
OBJECT_ONLINE    = 3

-- Classes
CLASS_DW = 0 .. CLASS_IK = 13

-- Maps
MAP_LORENCIA  = 0
MAP_DUNGEON   = 1
MAP_DEVIAS    = 2
MAP_NORIA     = 3
MAP_LOSTTOWER = 4
-- ... 140+ maps total
```

---

## `System/ScriptReader.lua`

Utility for reading custom flat-file configs (CSV/tab-delimited):

```lua
local Reader = require("System/ScriptReader")

local r = Reader:Load("MyPlugin/data.txt")
while true do
    local line = r:GetLine("END")
    if not line then break end

    local id    = r:GetAsNumber()
    local name  = r:GetAsString()
    local value = r:GetAsNumber()
end
r:Close()
```

---

## Plugin Layout Convention

Each plugin follows the same pattern:

```
MyPlugin/
├── MyPlugin.lua     ← main file, registered in ScriptMain.lua
└── Config.lua       ← settings table returned by require()
```

**MyPlugin.lua:**
```lua
local Config = require("MyPlugin/Config")

BridgeFunctionAttach("OnReadScript", function()
    if not Config.Switch then
        LogPrint("[MyPlugin] disabled")
        return
    end
    LogPrint("[MyPlugin] loaded")
end)

BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    if not Config.Switch then return end
    -- logic
end)
```

**Config.lua:**
```lua
return {
    Switch        = true,
    DatabaseTable = "myplugin_data",
    CooldownSecs  = 3600,
    Message       = "Hello!",
}
```

---

## `require()` Path Resolution

Paths are relative to the `LuaBridge/` root. The `.lua` extension is optional:

```lua
require("System/ScriptCore")          -- loads LuaBridge/System/ScriptCore.lua
require("MyPlugin/Config")            -- loads LuaBridge/MyPlugin/Config.lua
```

---

## Script Reload

The engine supports live script reload without restarting the server. When a reload is triggered:

1. `OnShutScript` fires on all registered handlers
2. Lua state is cleared
3. `ScriptMain.lua` is executed again
4. `OnReadScript` fires on all newly registered handlers

In-memory state (Lua tables, counters) is lost on reload. Persist important data to the database before reload if needed.
