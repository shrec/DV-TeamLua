# Server Callbacks

Callbacks let your plugin react to engine events. Register them with `BridgeFunctionAttach(eventName, handler)`. Multiple plugins can attach to the same event — all handlers are called in registration order.

```lua
BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    -- your code
end)
```

---

## Script Lifecycle

### `OnReadScript`
Called once when scripts are (re)loaded. Use this to initialize state, read config files, and print a load message.

```lua
BridgeFunctionAttach("OnReadScript", function()
    LogPrint("[MyPlugin] loaded")
    -- initialize your tables, read config files, etc.
end)
```

---

### `OnShutScript`
Called when the script engine shuts down (server stop or script reload before re-read).

```lua
BridgeFunctionAttach("OnShutScript", function()
    LogPrint("[MyPlugin] unloaded")
end)
```

---

## Timer

### `OnTimerThread`
Called every **1 second** by the engine. Use a counter to run logic at longer intervals.

```lua
local tick = 0

BridgeFunctionAttach("OnTimerThread", function()
    tick = tick + 1

    if tick % 60 == 0 then
        -- runs every 60 seconds
    end

    -- daily reset at midnight
    local t = os.date("*t")
    if t.hour == 0 and t.min == 0 and t.sec <= 1 then
        -- midnight logic
    end
end)
```

---

## Character Events

### `OnCharacterEntry(aIndex)`
Fires when a character finishes loading and appears in the world.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |

```lua
BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
    NoticeSend(aIndex, 0, "Welcome, " .. GetObjectName(aIndex) .. "!")
end)
```

---

### `OnCharacterClose(aIndex)`
Fires when a character disconnects or logs out.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |

```lua
BridgeFunctionAttach("OnCharacterClose", function(aIndex)
    local name = GetObjectName(aIndex)
    -- save any unsaved data for this player
end)
```

---

## Combat Events

### `OnMonsterDie(aIndex, bIndex, skill)`
Fires when a monster is killed.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index (killer) |
| `bIndex` | integer | Monster index |
| `skill` | integer | Skill ID used for the killing blow |

```lua
BridgeFunctionAttach("OnMonsterDie", function(aIndex, bIndex, skill)
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end

    local monsterType = GetObjectType(bIndex)
    local map = GetObjectMap(aIndex)
    -- custom drop logic, kill counter, etc.
end)
```

---

### `OnUserDie(aIndex, bIndex, skill)`
Fires when a player dies.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index (victim) |
| `bIndex` | integer | Killer index (player or monster) |
| `skill` | integer | Skill ID of the killing blow |

```lua
BridgeFunctionAttach("OnUserDie", function(aIndex, bIndex, skill)
    NoticeSend(aIndex, 0, "You have died.")
end)
```

---

### `OnUserRespawn(aIndex)`
Fires when a player respawns after death.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |

---

### `OnCheckUserTarget(aIndex, bIndex)`
Called to validate whether `aIndex` can attack `bIndex`. Return `0` to allow, `1` to block.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Attacker index |
| `bIndex` | integer | Target index |

**Returns:** `0` = allow, `1` = block attack

```lua
BridgeFunctionAttach("OnCheckUserTarget", function(aIndex, bIndex)
    -- prevent attacking in safe zone (example)
    if GetObjectMap(aIndex) == MAP_LORENCIA then
        return 1  -- blocked
    end
    return 0  -- allowed
end)
```

---

### `OnCheckUserKiller(aIndex, bIndex)`
Called after a kill to validate PK logic.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Killer index |
| `bIndex` | integer | Victim index |

**Returns:** `0` = allow, `1` = block

---

### `OnUserGiveExp(aIndex, exp)`
Fires whenever a player receives experience.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `exp` | integer | Amount of EXP gained |

---

## NPC & Economy Events

### `OnNpcTalk(aIndex, npcIndex)`
Fires when a player interacts with an NPC. Return `1` to handle and suppress the default NPC menu, `0` to let it pass through.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `npcIndex` | integer | NPC object index |

**Returns:** `1` = handled (suppress default), `0` = pass through

```lua
BridgeFunctionAttach("OnNpcTalk", function(aIndex, npcIndex)
    -- Custom NPC with type 450
    -- open your own dialog and return 1
    return 0
end)
```

---

### `OnUserBuyitem(aIndex, npcIndex, itemCat, itemIndex, itemCount)`
Fires when a player buys from an NPC store.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `npcIndex` | integer | NPC index |
| `itemCat` | integer | Item category |
| `itemIndex` | integer | Item index within category |
| `itemCount` | integer | Quantity |

**Returns:** `1` = allow, `0` = block purchase

---

### `OnUserSellitem(aIndex, npcIndex, slot)`
Fires when a player sells an item to an NPC.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `npcIndex` | integer | NPC index |
| `slot` | integer | Inventory slot of item being sold |

**Returns:** `1` = allow, `0` = block sale

---

## Item Events

### `OnUserItemPick(aIndex, itemIndex)`
Fires when a player picks up an item from the ground.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `itemIndex` | integer | Item object index on the map |

**Returns:** `1` = allow, `0` = block pickup

---

### `OnUserItemDrop(aIndex, slot)`
Fires when a player drops an item.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `slot` | integer | Inventory slot of dropped item |

**Returns:** `1` = allow, `0` = block drop

---

### `OnUserItemMove(aIndex, fromSlot, toSlot)`
Fires when a player moves an item within inventory.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `fromSlot` | integer | Source slot |
| `toSlot` | integer | Destination slot |

**Returns:** `1` = allow, `0` = block

---

### `OnUserItemUse(aIndex, slot)`
Fires when a player uses/consumes an item.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `slot` | integer | Inventory slot |

**Returns:** `1` = allow, `0` = block

---

### `OnUserEquipItem(aIndex, slot)`
Fires when a player equips an item.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `slot` | integer | Equipment slot |

**Returns:** `1` = allow, `0` = block

---

### `OnUserUnEquipItem(aIndex, slot)`
Fires when a player unequips an item.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `slot` | integer | Equipment slot |

**Returns:** `1` = allow, `0` = block

---

## Commands

### `OnCommandManager(aIndex, command)`
Fires when a player types a `/command`.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `command` | string | The command string (e.g. `"/reward"`) |

**Returns:** `1` = command was handled, `0` = not handled (pass to next handler)

Use `CommandGetArgString(n)` and `CommandGetArgNumber(n)` to read arguments.

```lua
BridgeFunctionAttach("OnCommandManager", function(aIndex, command)
    if command == "/coins" then
        local coins = ObjectGetCoin(aIndex, 1)
        NoticeSend(aIndex, 0, "You have " .. coins .. " coins.")
        return 1
    end
    return 0
end)
```

---

### `OnAdminCommandManager(aIndex, command)`
Same as `OnCommandManager` but only fires for Game Masters (`GetObjectAccountLevel(aIndex) >= 1`).

**Returns:** `1` = handled, `0` = pass through

---

### `OnCommandDone(aIndex, command)`
Fires after a command has been processed and accepted.

---

## Teleport & Map

### `OnUserTeleport(aIndex, map, x, y)`
Fires before a player teleports. Return `0` to allow, `1` to block.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `map` | integer | Destination map ID |
| `x` | integer | Destination X |
| `y` | integer | Destination Y |

**Returns:** `0` = allow, `1` = block teleport

---

## Event System

### `OnEventStart(eventId)`
Fires when a server event starts.

| Parameter | Type | Description |
|---|---|---|
| `eventId` | integer | Event type ID |

---

### `OnCanEnterEvent(aIndex, eventId)`
Called to check if a player can enter an event. Return `1` to allow, `0` to block.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `eventId` | integer | Event type ID |

**Returns:** `1` = allow, `0` = block

---

### `OnEventEnter(aIndex, eventId)`
Fires when a player enters an event.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `eventId` | integer | Event type ID |

---

## Network

### `OnPacketRecv(aIndex, headCode, subCode, data)`
Fires when a custom packet is received from the client.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `headCode` | integer | Packet head code |
| `subCode` | integer | Packet sub code |
| `data` | string | Raw packet payload |

---

## Database

### `OnSQLAsyncResult(label, rows)`
Fires with the result of a `SQLAsyncQuery()` call.

| Parameter | Type | Description |
|---|---|---|
| `label` | string | The label passed to `SQLAsyncQuery()` |
| `rows` | table | Array of row tables, each key is a column name |

```lua
BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if label ~= "myplugin_load" then return end

    if not rows[1] then
        -- no results
        return
    end

    local val = rows[1]["my_column"]
end)
```

See [Database Structures](Database-Structures) for the full SQL guide.
