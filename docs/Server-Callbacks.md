# Server Callbacks

Register handlers with `BridgeFunctionAttach(eventName, fn)`. Multiple plugins can attach to the same event.

```lua
BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    -- your code
end)
```

---

## Lifecycle

| Callback | When |
|---|---|
| `OnReadScript()` | Scripts loaded / reloaded |
| `OnShutScript()` | Script engine shutting down |
| `OnTimerThread()` | Every 1 second |

```lua
BridgeFunctionAttach("OnReadScript", function()
    LogPrint("[MyPlugin] loaded")
end)

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

### `OnCharacterEntry(aIndex)`
Character finishes loading and enters the world.
```lua
BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
    NoticeSend(aIndex, 0, "Welcome, " .. GetObjectName(aIndex) .. "!")
end)
```

### `OnCharacterClose(aIndex)`
Character disconnects or logs out.

---

## Combat

### `OnMonsterDie(aIndex, bIndex, skill)`
| Param | Description |
|---|---|
| `aIndex` | Killer (player) |
| `bIndex` | Monster index |
| `skill` | Skill ID of killing blow |

### `OnUserDie(aIndex, bIndex, skill)`
| Param | Description |
|---|---|
| `aIndex` | Victim (player) |
| `bIndex` | Killer index |
| `skill` | Skill ID |

### `OnUserRespawn(aIndex)`
Player respawns after death.

### `OnCheckUserTarget(aIndex, bIndex)` → 0/1
Validate attack. Return `1` to block, `0` to allow.

### `OnCheckUserKiller(aIndex, bIndex)` → 0/1
Validate kill. Return `1` to block PK consequence, `0` to allow.

### `OnUserGiveExp(aIndex, exp)`
Player receives experience.

---

## NPC & Economy

### `OnNpcTalk(aIndex, npcIndex)` → 0/1
Player clicks NPC. Return `1` to suppress default menu, `0` to pass through.

### `OnUserBuyitem(aIndex, npcIndex, itemCat, itemIndex, itemCount)` → 1/0
Return `1` to allow, `0` to block purchase.

### `OnUserSellitem(aIndex, npcIndex, slot)` → 1/0
Return `1` to allow, `0` to block sale.

---

## Items

| Callback | Params | Return |
|---|---|---|
| `OnUserItemPick(aIndex, itemIndex)` | player, item-on-ground | `1`=allow, `0`=block |
| `OnUserItemDrop(aIndex, slot)` | player, inv-slot | `1`=allow, `0`=block |
| `OnUserItemMove(aIndex, fromSlot, toSlot)` | player, slots | `1`=allow, `0`=block |
| `OnUserItemUse(aIndex, slot)` | player, inv-slot | `1`=allow, `0`=block |
| `OnUserEquipItem(aIndex, slot)` | player, equip-slot | `1`=allow, `0`=block |
| `OnUserUnEquipItem(aIndex, slot)` | player, equip-slot | `1`=allow, `0`=block |

---

## Commands

### `OnCommandManager(aIndex, command)` → 1/0
Player types `/command`. Return `1` = handled, `0` = pass through.

```lua
BridgeFunctionAttach("OnCommandManager", function(aIndex, command)
    if command ~= "/coins" then return 0 end
    NoticeSend(aIndex, 0, "Coins: " .. ObjectGetCoin(aIndex, 1))
    return 1
end)
```

### `OnAdminCommandManager(aIndex, command)` → 1/0
Same, but only fires for GM accounts.

### `OnCommandDone(aIndex, command)`
Fires after a command is accepted.

---

## Map & Events

### `OnUserTeleport(aIndex, map, x, y)` → 0/1
Before teleport. Return `1` to block.

### `OnEventStart(eventId)`
Server event starts.

### `OnCanEnterEvent(aIndex, eventId)` → 1/0
Return `1` to allow entry, `0` to block.

### `OnEventEnter(aIndex, eventId)`
Player enters event.

---

## Network & Database

### `OnPacketRecv(aIndex, headCode, subCode, data)`
Custom packet received from client.

### `OnSQLAsyncResult(label, rows)`
Result from `SQLAsyncQuery()`. `rows[n]["column"]` — all values are strings.

```lua
BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if label ~= "myplugin_load" then return end
    local val = tonumber(rows[1] and rows[1]["points"] or 0)
end)
```
