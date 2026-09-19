# Server Callbacks

Register handlers with `BridgeFunctionAttach(eventName, fn)`. Multiple plugins can attach to the same event — all are called in registration order.

> **Return value conventions**
> - Callbacks marked **"no return"** — the engine ignores whatever you return. You may omit `return`.
> - Callbacks marked **"return 1/0"** — the engine reads your return value and acts on it. Always `return` explicitly.

---

## Understanding aIndex and bIndex

Every player, monster, and NPC in the game is identified by an **integer index**. Callbacks give you these indices — they are not objects, just numbers. You pass them to the API functions to read any property.

```lua
-- aIndex is always the "actor" (attacker, buyer, moving player, etc.)
-- bIndex is always the "target" (monster, NPC, victim, etc.)

BridgeFunctionAttach("OnMonsterDie", function(aIndex, bIndex)
    -- aIndex = who killed it (player)
    -- bIndex = what died (monster)

    local killerName  = GetObjectName(aIndex)      -- "PlayerOne"
    local killerLevel = GetObjectLevel(aIndex)     -- 400
    local killerMap   = GetObjectMap(aIndex)       -- 0 (Lorencia)
    local killerClass = GetObjectClass(aIndex)     -- 1 (Dark Knight)

    local monsterMap  = GetObjectMap(bIndex)       -- same map
    local monsterX    = GetObjectMapX(bIndex)      -- X position
    local monsterY    = GetObjectMapY(bIndex)      -- Y position
end)
```

### Most common getters

```lua
GetObjectName(aIndex)          -- character name (string)
GetObjectLevel(aIndex)         -- level (integer)
GetObjectClass(aIndex)         -- class constant (CLASS_DW, CLASS_DK, …)
GetObjectReset(aIndex)         -- reset count
GetObjectMap(aIndex)           -- current map ID
GetObjectMapX(aIndex)          -- X coordinate
GetObjectMapY(aIndex)          -- Y coordinate
GetObjectMoney(aIndex)         -- Zen
GetObjectRuud(aIndex)          -- Ruud
GetObjectAccountLevel(aIndex)  -- VIP level (0=normal, 1–4=VIP)
GetObjectConnected(aIndex)     -- connection state (OBJECT_ONLINE = 3)
GetObjectType(aIndex)          -- OBJECT_USER / OBJECT_MONSTER / OBJECT_NPC
```

### Always validate before reading

```lua
-- Player might disconnect between event and your code running
if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end

-- Confirm the type before assuming it's a player
if GetObjectType(bIndex) ~= OBJECT_MONSTER then return end
```

Full list of getters: [Player Structure](Player-Structure.md)

---

## Lifecycle

### `OnReadScript()` — no return

Called once when scripts finish loading (or after a live reload). Initialize tables, read config files, print a ready message.

```lua
BridgeFunctionAttach("OnReadScript", function()
    LogPrint("[MyPlugin] loaded")
    -- initialize plugin state here
end)
```

---

### `OnShutScript()` — no return

Called when the script engine shuts down (server stop, or just before a live reload).

```lua
BridgeFunctionAttach("OnShutScript", function()
    -- flush anything you need to save
end)
```

---

### `OnTimerThread()` — no return

Called every **1 second**. Use a counter to run code at longer intervals.

```lua
local tick = 0
BridgeFunctionAttach("OnTimerThread", function()
    tick = tick + 1
    if tick % 60 == 0 then
        -- every 60 seconds
    end
end)
```

---

## Character

### `OnCharacterEntry(aIndex)` — no return

Character finishes loading and appears in the world.

| Parameter | Type | Value |
|---|---|---|
| `aIndex` | integer | Player index |

```lua
BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
    NoticeSend(aIndex, 0, "Welcome, " .. GetObjectName(aIndex) .. "!")
end)
```

---

### `OnCharacterClose(aIndex)` — no return

Character disconnects or logs out. Last chance to save player data.

| Parameter | Type | Value |
|---|---|---|
| `aIndex` | integer | Player index |

```lua
BridgeFunctionAttach("OnCharacterClose", function(aIndex)
    local name = GetObjectName(aIndex)
    -- flush unsaved data for this player
end)
```

---

## Commands

### `OnCommandManager(aIndex, Type, code, arg)` — return 1 or 0

Called when a player types any `/command`.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `Type` | integer | Command type (internal classification) |
| `code` | integer | Numeric command code |
| `arg` | string | Raw argument string after the command |

**Return `1`** — command was handled by your plugin, stop processing.  
**Return `0`** — not handled, pass to next handler.

Use `CommandGetArgString(n)` / `CommandGetArgNumber(n)` to parse individual arguments.

```lua
BridgeFunctionAttach("OnCommandManager", function(aIndex, Type, code, arg)
    if code ~= 100 then return 0 end  -- your custom command code

    local amount = CommandGetArgNumber(1)
    ObjectAddCoin(aIndex, 1, amount)
    NoticeSend(aIndex, 0, "Added " .. amount .. " coins.")
    return 1
end)
```

---

### `OnAdminCommandManager(aIndex, Type, code, arg)` — return 1 or 0

Same as `OnCommandManager` but only fires for accounts with GM level ≥ 1. Same parameters and return convention.

```lua
BridgeFunctionAttach("OnAdminCommandManager", function(aIndex, Type, code, arg)
    if code ~= 999 then return 0 end
    local targetName = CommandGetArgString(1)
    -- admin action...
    return 1
end)
```

---

### `OnCommandDone(aIndex, code)` — no return

Fires after a command was accepted and executed.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `code` | integer | Command code that completed |

---

## Combat

### `OnMonsterDie(aIndex, bIndex)` — no return

A monster was killed.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Killer player index |
| `bIndex` | integer | Monster index |

> ⚠️ No skill parameter is passed — only killer and monster indices.

```lua
BridgeFunctionAttach("OnMonsterDie", function(aIndex, bIndex)
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
    if GetObjectType(bIndex) ~= OBJECT_MONSTER then return end

    local map = GetObjectMap(bIndex)
    local x   = GetObjectMapX(bIndex)
    local y   = GetObjectMapY(bIndex)
    -- custom drop logic
    ItemDrop(map, x, y, 13, 14, 0, aIndex)
end)
```

---

### `OnUserDie(aIndex, bIndex)` — no return

A player died.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Victim player index |
| `bIndex` | integer | Killer index (player or monster) |

```lua
BridgeFunctionAttach("OnUserDie", function(aIndex, bIndex)
    NoticeSend(aIndex, 0, "You died. Respawning...")
end)
```

---

### `OnUserRespawn(aIndex, killerType)` — no return

Player respawns after death.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `killerType` | integer | Type of entity that killed them (`OBJECT_USER`, `OBJECT_MONSTER`, etc.) |

```lua
BridgeFunctionAttach("OnUserRespawn", function(aIndex, killerType)
    if killerType == OBJECT_USER then
        NoticeSend(aIndex, 0, "You were killed by another player.")
    end
end)
```

---

### `OnUserGiveExp(aIndex, exp)` — no return

Player receives experience.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `exp` | number | Amount of EXP gained (can be large — int64) |

---

### `OnCheckUserTarget(aIndex, bIndex)` — return 1 or 0

Called every time a player attempts to attack a target. Runs frequently — keep it fast.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Attacker index |
| `bIndex` | integer | Target index |

**Return `1`** — allow the attack.  
**Return `0`** — block the attack.

> Default when no handler or on error: `1` (allow).

```lua
BridgeFunctionAttach("OnCheckUserTarget", function(aIndex, bIndex)
    -- Block attacks in Lorencia
    if GetObjectMap(aIndex) == MAP_LORENCIA then
        return 0  -- blocked
    end
    return 1  -- allowed
end)
```

---

### `OnCheckUserKiller(aIndex, bIndex)` — return 1 or 0

Called after a kill to validate PK consequences.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Killer index |
| `bIndex` | integer | Victim index |

**Return `1`** — allow PK consequence (PK count increases, etc.).  
**Return `0`** — suppress PK consequence.

> Default when no handler or on error: `1` (allow).

```lua
BridgeFunctionAttach("OnCheckUserKiller", function(aIndex, bIndex)
    -- No PK penalty in event maps
    if GetObjectMap(aIndex) == MAP_ARENA then
        return 0
    end
    return 1
end)
```

---

## NPC & Economy

### `OnNpcTalk(aIndex, bIndex)` — return 1 or 0

Player clicks an NPC.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `bIndex` | integer | NPC object index |

**Return `1`** — handled by Lua, suppress the default NPC menu.  
**Return `0`** — not handled, show the default NPC menu.

```lua
BridgeFunctionAttach("OnNpcTalk", function(aIndex, bIndex)
    local map = GetObjectMap(bIndex)
    local x   = GetObjectMapX(bIndex)
    local y   = GetObjectMapY(bIndex)

    if map == MAP_LORENCIA and x == 125 and y == 125 then
        ChatTargetSend(aIndex, bIndex, "Hello! Buy coins here.")
        return 1  -- suppress default menu
    end
    return 0
end)
```

---

### `OnUserBuyitem(aIndex, npcIndex, itemTable)` — return 1 or 0

Player attempts to buy an item from an NPC store.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `npcIndex` | integer | NPC index |
| `itemTable` | table | Item being purchased (same fields as inventory item table) |

**Return `1`** — allow the purchase.  
**Return `0`** — block the purchase.

```lua
BridgeFunctionAttach("OnUserBuyitem", function(aIndex, npcIndex, itemTable)
    -- Block buying wings if player has < 10 resets
    if itemTable.ItemCat == 12 and GetObjectReset(aIndex) < 10 then
        NoticeSend(aIndex, 0, "Need 10 resets to buy wings.")
        return 0
    end
    return 1
end)
```

---

### `OnUserSellitem(aIndex, npcIndex, itemTable, itemSlot)` — return 1 or 0

Player attempts to sell an item to an NPC.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `npcIndex` | integer | NPC index |
| `itemTable` | table | Item being sold |
| `itemSlot` | integer | Inventory slot of the item |

**Return `1`** — allow the sale.  
**Return `0`** — block the sale.

```lua
BridgeFunctionAttach("OnUserSellitem", function(aIndex, npcIndex, itemTable, itemSlot)
    -- Prevent selling ancient items
    if itemTable.AncientOpt and itemTable.AncientOpt > 0 then
        NoticeSend(aIndex, 0, "Cannot sell ancient items.")
        return 0
    end
    return 1
end)
```

---

## Items

### `OnUserItemPick(aIndex, itemTable)` — return 1 or 0

Player attempts to pick up an item from the ground.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `itemTable` | table | Item on the ground (full item table with all fields) |

**Return `1`** — allow pickup.  
**Return `0`** — block pickup.

```lua
BridgeFunctionAttach("OnUserItemPick", function(aIndex, itemTable)
    -- Block picking up wings in non-wing zones (example)
    if itemTable.ItemCat == 12 then
        return 0
    end
    return 1
end)
```

---

### `OnUserItemDrop(aIndex, slot, x, y, itemTable)` — return 1 or 0

Player attempts to drop an item.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `slot` | integer | Inventory slot of the item being dropped |
| `x` | integer | Target drop X coordinate on map |
| `y` | integer | Target drop Y coordinate on map |
| `itemTable` | table | Item being dropped |

**Return `1`** — allow the drop.  
**Return `0`** — block the drop.

```lua
BridgeFunctionAttach("OnUserItemDrop", function(aIndex, slot, x, y, itemTable)
    -- Prevent dropping items in safe zones
    if MapCheckAttr(GetObjectMap(aIndex), x, y, 1) then
        NoticeSend(aIndex, 0, "Cannot drop items here.")
        return 0
    end
    return 1
end)
```

---

### `OnUserItemMove(aIndex, aFlag, aSlot, bFlag, bSlot, state)` — return 1 or 0

Player moves an item within inventory (drag from one slot to another).

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `aFlag` | integer | Source inventory type (0=main, 1=event, etc.) |
| `aSlot` | integer | Source slot |
| `bFlag` | integer | Destination inventory type |
| `bSlot` | integer | Destination slot |
| `state` | integer | Move state/type |

**Return `1`** — allow the move.  
**Return `0`** — block the move.

---

### `OnUserItemUse(aIndex, sourceSlot, targetSlot, useType)` — return 1 or 0

Player uses/activates an item.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `sourceSlot` | integer | Slot of the item being used |
| `targetSlot` | integer | Target slot (e.g. for combining items) |
| `useType` | integer | Use action type |

**Return `1`** — allow the use.  
**Return `0`** — block the use.

---

### `OnUserEquipItem(aIndex, sourceSlot, targetSlot, state)` — return 1 or 0

Player equips an item from inventory.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `sourceSlot` | integer | Bag slot the item is coming from |
| `targetSlot` | integer | Equipment slot being equipped to |
| `state` | integer | Equip state |

**Return `1`** — allow equip.  
**Return `0`** — block equip.

```lua
BridgeFunctionAttach("OnUserEquipItem", function(aIndex, sourceSlot, targetSlot, state)
    local item = InventoryGetItemTable(aIndex, sourceSlot)
    if not item then return 1 end

    -- Block equipping wings below reset 5
    if item.ItemCat == 12 and GetObjectReset(aIndex) < 5 then
        NoticeSend(aIndex, 0, "Need 5 resets to wear wings.")
        return 0
    end
    return 1
end)
```

---

### `OnUserUnEquipItem(aIndex, sourceSlot, targetSlot, state)` — return 1 or 0

Player removes an equipped item.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `sourceSlot` | integer | Equipment slot being unequipped |
| `targetSlot` | integer | Bag slot destination |
| `state` | integer | Unequip state |

**Return `1`** — allow unequip.  
**Return `0`** — block unequip.

---

## Map & Teleport

### `OnUserTeleport(aIndex, map, gate)` — return 1 or 0

Player attempts to teleport through a gate.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `map` | integer | Destination map ID |
| `gate` | integer | Gate ID being used |

**Return `1`** — allow teleport.  
**Return `0`** — block teleport.

> ⚠️ Third parameter is the **gate ID**, not X/Y coordinates.

```lua
BridgeFunctionAttach("OnUserTeleport", function(aIndex, map, gate)
    -- Block entry to map 2 (Devias) if reset < 1
    if map == MAP_DEVIAS and GetObjectReset(aIndex) < 1 then
        NoticeSend(aIndex, 0, "Need 1 reset to enter Devias.")
        return 0
    end
    return 1
end)
```

---

## Events

### `OnEventStart(eventId)` — no return

A server event starts.

| Parameter | Type | Description |
|---|---|---|
| `eventId` | integer | Event type ID |

---

### `OnCanEnterEvent(aIndex, eventId)` — return 1 or 0

Called to check if a player may enter an event.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `eventId` | integer | Event type ID |

**Return `1`** — allow entry.  
**Return `0`** — block entry.

---

### `OnEventEnter(aIndex, eventId)` — no return

Player successfully enters an event.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `eventId` | integer | Event type ID |

---

## Network

### `OnPacketRecv(aIndex, data, size)` — no return

Custom packet received from the client.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Player index |
| `data` | string | Raw binary packet data (use `string.byte` to parse) |
| `size` | integer | Packet size in bytes |

---

## Database

### `OnSQLAsyncResult(label, callbackParam, rows)` — no return

Result of a `SQLAsyncQuery()` call. The callback always receives three arguments,
even when the optional `callbackParam` was not supplied.

| Parameter | Type | Description |
|---|---|---|
| `label` | string | Label passed to `SQLAsyncQuery()` |
| `callbackParam` | string | Optional context supplied as the third argument to `SQLAsyncQuery()` |
| `rows` | table or number | SELECT row table; otherwise affected-row count or `0` for no result. Numeric SQL fields become Lua numbers; other non-NULL fields become strings. |

```lua
BridgeFunctionAttach("OnSQLAsyncResult", function(label, callbackParam, rows)
    if label ~= "myplugin_load" then return end
    if type(rows) ~= "table" or not rows[1] then return end

    local points = tonumber(rows[1]["points"]) or 0
end)
```

A failed SELECT and a successful empty SELECT both reach the legacy callback
without a distinct error status. Do not interpret `rows == 0` as proof of success.

---

## Quick Reference

| Callback | Parameters | Return |
|---|---|---|
| `OnReadScript` | — | none |
| `OnShutScript` | — | none |
| `OnTimerThread` | — | none |
| `OnCharacterEntry` | `aIndex` | none |
| `OnCharacterClose` | `aIndex` | none |
| `OnCommandManager` | `aIndex, Type, code, arg` | `1`=handled `0`=pass |
| `OnAdminCommandManager` | `aIndex, Type, code, arg` | `1`=handled `0`=pass |
| `OnCommandDone` | `aIndex, code` | none |
| `OnMonsterDie` | `aIndex, bIndex` | none |
| `OnUserDie` | `aIndex, bIndex` | none |
| `OnUserRespawn` | `aIndex, killerType` | none |
| `OnUserGiveExp` | `aIndex, exp` | none |
| `OnCheckUserTarget` | `aIndex, bIndex` | `1`=allow `0`=block |
| `OnCheckUserKiller` | `aIndex, bIndex` | `1`=allow `0`=block |
| `OnNpcTalk` | `aIndex, npcIndex` | `1`=handled `0`=default |
| `OnUserBuyitem` | `aIndex, npcIndex, itemTable` | `1`=allow `0`=block |
| `OnUserSellitem` | `aIndex, npcIndex, itemTable, slot` | `1`=allow `0`=block |
| `OnUserItemPick` | `aIndex, itemTable` | `1`=allow `0`=block |
| `OnUserItemDrop` | `aIndex, slot, x, y, itemTable` | `1`=allow `0`=block |
| `OnUserItemMove` | `aIndex, aFlag, aSlot, bFlag, bSlot, state` | `1`=allow `0`=block |
| `OnUserItemUse` | `aIndex, sourceSlot, targetSlot, useType` | `1`=allow `0`=block |
| `OnUserEquipItem` | `aIndex, srcSlot, dstSlot, state` | `1`=allow `0`=block |
| `OnUserUnEquipItem` | `aIndex, srcSlot, dstSlot, state` | `1`=allow `0`=block |
| `OnUserTeleport` | `aIndex, map, gate` | `1`=allow `0`=block |
| `OnEventStart` | `eventId` | none |
| `OnCanEnterEvent` | `aIndex, eventId` | `1`=allow `0`=block |
| `OnEventEnter` | `aIndex, eventId` | none |
| `OnPacketRecv` | `aIndex, data, size` | none |
| `OnSQLAsyncResult` | `label, callbackParam, rows` | none |
