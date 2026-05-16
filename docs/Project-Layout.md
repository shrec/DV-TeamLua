# Project Layout

## Directory Structure

```
LuaBridge/
├── ScriptMain.lua                  ← engine entry point
├── System/
│   ├── ScriptCore.lua              ← BridgeFunctionAttach + callback dispatch
│   ├── ScriptDefine.lua            ← constants (classes, maps, object types)
│   └── ScriptReader.lua            ← flat-file parser utility
└── MyPlugin/
    ├── MyPlugin.lua
    └── Config.lua
```

## Entry Point — `ScriptMain.lua`

```lua
require("System/ScriptCore")
require("DailyReward/DailyReward")
require("WelcomeMessage/Script")
require("MyPlugin/MyPlugin")
-- comment out to disable a plugin:
-- require("MyPlugin/MyPlugin")
```

## Plugin Convention

**MyPlugin.lua:**
```lua
local Config = require("MyPlugin/Config")

BridgeFunctionAttach("OnReadScript", function()
    LogPrint("[MyPlugin] loaded")
end)

BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    if not Config.Switch then return end
end)
```

**Config.lua:**
```lua
return {
    Switch        = true,
    DatabaseTable = "myplugin_data",
    CooldownSecs  = 3600,
}
```

## `require()` Path

Paths are relative to `LuaBridge/`:
```lua
require("System/ScriptCore")    -- LuaBridge/System/ScriptCore.lua
require("MyPlugin/Config")      -- LuaBridge/MyPlugin/Config.lua
```

## Script Reload

On reload: `OnShutScript` fires → Lua state cleared → `ScriptMain.lua` re-executed → `OnReadScript` fires. All in-memory tables are reset.

## `System/ScriptReader.lua`

Parse custom flat-file configs:
```lua
local Reader = require("System/ScriptReader")
local r = Reader:Load("MyPlugin/data.txt")
while true do
    local line = r:GetLine("END")
    if not line then break end
    local id    = r:GetAsNumber()
    local name  = r:GetAsString()
end
r:Close()
```
