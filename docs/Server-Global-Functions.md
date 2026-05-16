# Server Global Functions

Complete reference for every C++ function available in server-side Lua. All are globals — no `require` needed.

> **Index parameters** — `aIndex` / `bIndex` are integer player/monster indices. See [Server Callbacks](Server-Callbacks.md) for how to obtain them.

---

## Server Info

```lua
GetGameServerLang()       -- server language code (integer)
GetGameServerCode()       -- server ID / code
GetGameServerVersion()    -- version number
GetGameServerProtocol()   -- protocol version
GetGameServerCurUser()    -- current online player count
GetGameServerMaxUser()    -- server player cap
GetMaxIndex()             -- max object index
GetMinUserIndex()         -- first player index
GetMaxUserIndex()         -- last player index
GetMinMonsterIndex()      -- first monster/NPC index
GetMaxMonsterIndex()      -- last monster/NPC index
```

```lua
local count = GetGameServerCurUser()
SetWindowText("MyServer | Online: " .. count)
```

---

## Player — Getters

See [Player Structure](Player-Structure.md) for the full list. Quick reference:

```lua
-- Identity
GetObjectConnected(aIndex)     -- 0=offline 1=connected 2=logged 3=online
GetObjectType(aIndex)          -- OBJECT_USER/MONSTER/NPC/ITEM
GetObjectName(aIndex)          -- character name (string)
GetObjectAccount(aIndex)       -- account name
GetObjectAccountID(aIndex)     -- account DB ID
GetObjectIpAddress(aIndex)     -- IP address string
GetObjectHardwareId(aIndex)    -- hardware fingerprint
GetObjectPlayerID(aIndex)      -- internal player GUID
GetObjectPersonalCode(aIndex)  -- personal code
GetObjectIndexByName(name)     -- find index by character name (-1 if offline)
GetPlayerMac(aIndex)           -- MAC address
GetPlayerSerial(aIndex)        -- disk serial

-- Class & Level
GetObjectClass(aIndex)         -- base class (CLASS_DW … CLASS_IK)
GetObjectChangeUp(aIndex)      -- tier: 0=base 1=2nd 2=3rd
GetObjectLevel(aIndex)
GetObjectMasterLevel(aIndex)
GetObjectMajesticLevel(aIndex)
GetObjectToTalLevel(aIndex)    -- combined total
GetObjectLevelUpPoint(aIndex)  -- available stat points
GetObjectMasterPoint(aIndex)
GetObjectMajesticPoint(aIndex)
GetObjectReset(aIndex)
GetObjectMasterReset(aIndex)

-- Stats
GetObjectStrength/Dexterity/Vitality/Energy/Leadership(aIndex)        -- base
GetObjectExtraStrength/Dexterity/Vitality/Energy/Leadership(aIndex)   -- bonus
GetObjectTotalStrength/Dexterity/Vitality/Energy/Leadership(aIndex)   -- total
GetObjectDefaultStrength/Dexterity/Vitality/Energy/Leadership(aIndex) -- default

-- HP / Mana / BP / Shield
GetObjectLive(aIndex)          -- boolean
GetObjectLife(aIndex)          GetObjectMaxLife(aIndex)
GetObjectMana(aIndex)          GetObjectMaxMana(aIndex)
GetObjectBP(aIndex)            GetObjectMaxBP(aIndex)
GetObjectShield(aIndex)        GetObjectMaxShield(aIndex)

-- Money
GetObjectMoney(aIndex)         -- Zen
GetObjectRuud(aIndex)

-- Map
GetObjectMap(aIndex)    GetObjectMapX(aIndex)    GetObjectMapY(aIndex)
GetObjectDeathMap(aIndex) GetObjectDeathMapX(aIndex) GetObjectDeathMapY(aIndex)

-- Social
GetObjectPartyNumber(aIndex)
GetObjectGuildNumber(aIndex)   GetObjectGuildName(aIndex)
GetObjectGuildStatus(aIndex)   GetObjectGuildRelationship(aIndex, bIndex)
GetObjectGuildUnionNumber(aIndex) GetObjectGuildUnionName(aIndex)
GetObjectCSGuildSide(aIndex)

-- PK / Gens
GetObjectPKCount(aIndex)   GetObjectPKLevel(aIndex)   GetObjectPKTimer(aIndex)
GetObjectGensFamily(aIndex) GetObjectGensRank(aIndex)
GetObjectGensSymbol(aIndex) GetObjectGensContribution(aIndex)

-- Account / VIP
GetObjectAccountLevel(aIndex)       -- 0=normal 1-4=VIP tiers
GetObjectAccountExpireDate(aIndex)  -- expiry date string
GetObjectOfflineFlag(aIndex)
GetObjectAuthority(aIndex)
GetObjectInterface(aIndex)          -- open UI window ID
GetObjectChange(aIndex)
GetObjectLastGainedExp(aIndex)
GetObjectMaxNormalPoints(aIndex)
GetObjectLang(aIndex)               -- client language
```

---

## Player — Setters

After changing stats, call `UserCalcAttribute(aIndex)` to recalculate derived values.

```lua
SetObjectLevel(aIndex, value)
SetObjectLevelUpPoint(aIndex, value)
SetObjectMoney(aIndex, value)
SetObjectRuud(aIndex, value)
SetObjectStrength(aIndex, value)
SetObjectDexterity(aIndex, value)
SetObjectVitality(aIndex, value)
SetObjectEnergy(aIndex, value)
SetObjectLeadership(aIndex, value)
SetObjectMasterLevel(aIndex, value)
SetObjectMasterPoint(aIndex, value)
SetObjectMajesticLevel(aIndex, value)
SetObjectMajesticPoint(aIndex, value)
SetObjectLife(aIndex, value)        SetObjectMaxLife(aIndex, value)
SetObjectMana(aIndex, value)        SetObjectMaxMana(aIndex, value)
SetObjectBP(aIndex, value)          SetObjectMaxBP(aIndex, value)
SetObjectShield(aIndex, value)      SetObjectMaxShield(aIndex, value)
SetObjectPKCount(aIndex, value)
SetObjectPKLevel(aIndex, value)
SetObjectPKTimer(aIndex, value)
SetObjectLive(aIndex, value)        -- 0=dead 1=alive
SetObjectReset(aIndex, value)
SetObjectMap(aIndex, value)
SetObjectMapX(aIndex, value)
SetObjectMapY(aIndex, value)
SetObjectChatLimitTime(aIndex, seconds)   -- mute for N seconds
ChangeObjectReset(aIndex)                 -- trigger full reset process
```

```lua
-- Give 100 STR and recalculate
SetObjectStrength(aIndex, GetObjectStrength(aIndex) + 100)
UserCalcAttribute(aIndex)
LevelUpSend(aIndex)
```

---

## Player — Actions & Sync

### `UserCalcAttribute(aIndex)`
Recalculate all derived stats after changing base values. Always call after setters.

### `UserInfoSend(aIndex)`
Resend character info packet to client (refreshes displayed stats).

### `LevelUpSend(aIndex)`
Play level-up animation and sync level to client.

### `MasterLevelUpSend(aIndex)`
Play master level-up animation.

### `MasterSkillTreeRebuild(aIndex, group)`
Rebuild the master skill tree for the given group.

| Parameter | Description |
|---|---|
| `aIndex` | Player index |
| `group` | Skill group to rebuild |

### `UpdatePlayerLevels(aIndex)`
Sync all level data to the client at once.

### `CalculateCharacter(aIndex)`
Full server-side character recalculation (heavier than `UserCalcAttribute`).

### `ResetMajesticTree(aIndex)`
Reset the player's majestic skill tree.

### `PKLevelSend(aIndex, level)`
Set and sync PK level to client.

### `UserDisconnect(aIndex)`
Force disconnect the player's socket immediately.

### `UserGameLogout(aIndex, type)`
Graceful game logout.

| `type` | Effect |
|---|---|
| `0` | Normal logout |
| `1` | Kick to character select |

```lua
UserGameLogout(aIndex, 1)  -- kick to character select
```

### `UserWarehouseOpen(aIndex [, warehouseIndex])`
Open the warehouse UI for a player. Optional second argument selects a specific warehouse.

```lua
UserWarehouseOpen(aIndex)     -- default warehouse
UserWarehouseOpen(aIndex, 2)  -- second warehouse slot
```

### `SkinChangeSend(aIndex, skinId)`
Override player's visual skin/appearance.

### `UserActionSend(aIndex, targetIndex, action)`
Force `aIndex` to perform an animation toward `targetIndex`.

### `UserSetAccountLevel(aIndex, level)`
Set VIP account level (0=normal, 1-4=VIP).

---

## Teleport

### `MoveUser(aIndex, gate)`
Teleport player through a gate by gate ID.

```lua
MoveUser(aIndex, 1)   -- gate 1
```

### `MoveUserEx(aIndex, world, x, y)`
Teleport to exact map coordinates.

```lua
MoveUserEx(aIndex, MAP_LORENCIA, 125, 125)
```

---

## Messaging

### `NoticeSend(aIndex, type, message)`
Send an in-game notice to one player.

| `type` | Appearance |
|---|---|
| `0` | Normal (white) |
| `1` | Blue |
| `2` | Guild color |
| `3` | Red |

```lua
NoticeSend(aIndex, 0, "Server restarts in 5 minutes.")
NoticeSend(aIndex, 3, "Warning: you are in a PK zone!")
```

### `NoticeSendToAll(type, message)`
Broadcast to all online players.

```lua
NoticeSendToAll(1, "Castle Siege starts now!")
```

### `NoticeGlobalSend(type, message)`
Server-wide notice across all GS instances (multi-server setup).

### `MessageSend(aIndex, type, title, body)`
Send a popup dialog with title and body text.

```lua
MessageSend(aIndex, 0, "Daily Reward", "You received 100 coins!")
```

### `MessageSendToAll(type, title, body)`
Popup dialog to all players.

### `ChatTargetSend(npcIndex, aIndex, message)`
Send a chat message appearing to come from an NPC. Shows NPC dialogue bubble.

> ⚠️ First parameter is the **NPC index**, second is the **player index**.

```lua
BridgeFunctionAttach("OnNpcTalk", function(aIndex, npcIndex)
    ChatTargetSend(npcIndex, aIndex, "Welcome, adventurer!")
    return 1
end)
```

### `SendRgbChat(aIndex, r, g, b, message)`
Send a colored chat message.

```lua
SendRgbChat(aIndex, 255, 215, 0, "You found a legendary item!")  -- gold text
```

### `PostSend(type, messageId, name, text)`
Send a postal / inbox message.

### `LogPrint(message)`
Write to server log (console + log file).

```lua
LogPrint("[MyPlugin] Player " .. GetObjectName(aIndex) .. " used /reward")
```

### `LogColor(color, message)`
Write a colored message to console (color = Windows FOREGROUND constant).

```lua
LogColor(12, "[ERROR] Something went wrong")  -- 12 = red
```

---

## Commands

### `CommandCheckGameMasterLevel(aIndex, level)` → boolean
Return `true` if the player's GM account level meets `level`.

```lua
if not CommandCheckGameMasterLevel(aIndex, 1) then
    NoticeSend(aIndex, 0, "No permission.")
    return 1
end
```

### `CommandGetArgNumber(n)` → integer
Get the Nth argument of the current command as a number (1-indexed).

```lua
-- /addcoins 500
local amount = CommandGetArgNumber(1)  -- 500
```

### `CommandGetArgString(n)` → string
Get the Nth argument as a string.

```lua
-- /kick PlayerName
local name = CommandGetArgString(1)   -- "PlayerName"
```

---

## Experience & Levels

### `ExpSend(aIndex, amount)`
Give experience points and update the client.

```lua
ExpSend(aIndex, 5000000)
```

### `LevelUpSend(aIndex)`
Trigger level-up visual effect and sync.

### `MasterLevelUpSend(aIndex)`
Trigger master level-up visual effect.

### `RuudSend(aIndex, amount)`
Give Ruud and sync balance to client.

```lua
RuudSend(aIndex, 1000)
```

---

## Currency

### `ObjectGetCoin(aIndex)` → table
Returns a table with coin balances.

```lua
local coins = ObjectGetCoin(aIndex)
local credits    = coins["Coin1"]   -- credit coins
local goblin     = coins["Coin2"]   -- goblin points
```

### `ObjectAddCoin(aIndex, credit, goblin, cValue)`
Add coins.

```lua
ObjectAddCoin(aIndex, 100, 0, 0)   -- add 100 credits
ObjectAddCoin(aIndex, 0, 50, 0)    -- add 50 goblin points
```

### `ObjectSubCoin(aIndex, credit, goblin, cValue)`
Subtract coins.

```lua
ObjectSubCoin(aIndex, 100, 0, 0)   -- spend 100 credits
```

### `CashShopGetPoint(aIndex)` → table
Returns cash shop balances.

```lua
local cash = CashShopGetPoint(aIndex)
local wcoinC    = cash["WCoinC"]
local wcoinP    = cash["WCoinP"]
local goblin    = cash["GoblinPoint"]
```

### `CashShopAddPoint(aIndex, wcoinC, wcoinP, goblin)`
Add cash shop points.

```lua
CashShopAddPoint(aIndex, 500, 0, 0)   -- add 500 WCoinC
```

### `CashShopSubPoint(aIndex, wcoinC, wcoinP, goblin)`
Subtract cash shop points.

---

## Effects / Buffs

### `EffectAdd(aIndex, flag, buffId, duration, value1, value2, value3, value4)`
Apply a buff effect to a player.

| Parameter | Description |
|---|---|
| `aIndex` | Player index |
| `flag` | Buff category flag |
| `buffId` | Buff ID (internal constant) |
| `duration` | Duration in seconds |
| `value1–4` | Buff strength values (buff-specific) |

```lua
EffectAdd(aIndex, 0, 100, 30, 10, 0, 0, 0)   -- buff ID 100, 30 seconds, value=10
```

### `EffectDel(aIndex, buffId)`
Remove a specific buff.

```lua
EffectDel(aIndex, 100)
```

### `EffectCheck(aIndex, buffId)` → boolean
Check if player currently has a buff.

```lua
if EffectCheck(aIndex, 100) then
    NoticeSend(aIndex, 0, "You already have this buff.")
    return 1
end
```

### `EffectClear(aIndex)`
Remove all buffs from a player.

---

## Map & World

### `MapCheckAttr(map, x, y, attr)` → boolean
Check a tile attribute at map coordinates.

| `attr` | Meaning |
|---|---|
| `0` | Walkable? |
| `1` | Safe zone? |
| `2` | No-PK zone? |

```lua
if MapCheckAttr(GetObjectMap(aIndex), GetObjectMapX(aIndex), GetObjectMapY(aIndex), 1) then
    NoticeSend(aIndex, 0, "You are in a safe zone.")
end
```

### `GetMapName(mapId)` → string
Returns the display name of a map.

```lua
local name = GetMapName(MAP_LORENCIA)   -- "Lorencia"
```

### `MapGetItemTable(mapId, x)` → table
Get item data from the world at a position.

### `FireworksSend(aIndex, x, y)`
Trigger a fireworks particle effect at coordinates on the player's current map.

```lua
FireworksSend(aIndex, GetObjectMapX(aIndex), GetObjectMapY(aIndex))
```

### `SetWindowText(aIndex, title)`
Set the game server window title. If `aIndex` is invalid, broadcasts to all windows.

```lua
SetWindowText(-1, "MyServer | " .. GetGameServerCurUser() .. " online")
```

---

## Permissions

Each permission ID gates a specific player action:

| ID | Permission |
|---|---|
| 1 | MoveItems |
| 2 | SellItem |
| 3 | BuyItem |
| 4 | UseItem |
| 5 | DropItem |
| 6 | PickItem |
| 7 | OpenTrade |
| 8 | OpenPersonalShop |
| 9 | UseChaosMachine |
| 10 | OpenCashShop |
| 11 | UseChat |
| 12 | Teleport |
| 13 | MoveCharacter |

### `PermissionCheck(aIndex, permId)` → boolean
```lua
if not PermissionCheck(aIndex, 11) then
    -- player is muted
end
```

### `PermissionInsert(aIndex, permId)`
Grant a permission.
```lua
PermissionInsert(aIndex, 11)   -- unmute
```

### `PermissionRemove(aIndex, permId)`
Revoke a permission.
```lua
PermissionRemove(aIndex, 11)   -- mute (disable chat)
```

---

## Party

### `PartyCreate(aIndex)` → 1/0
Create a party with `aIndex` as leader. Returns `1` on success.

```lua
local ok = PartyCreate(aIndex)
```

### `PartyDelete(partyId)`
Disband a party.

### `PartyAddMember(partyId, aIndex)` → 1/0
Add a player to a party.

### `PartyDelMember(partyId, aIndex)`
Remove a player from a party.

### `PartyGetMemberCount(partyId)` → integer
Get number of members.

### `PartyGetMemberIndex(partyId, slot)` → integer
Get the player index at party slot (0-based).

```lua
-- Send notice to all party members
local partyId = GetObjectPartyNumber(aIndex)
if partyId >= 0 then
    local count = PartyGetMemberCount(partyId)
    for i = 0, count - 1 do
        local member = PartyGetMemberIndex(partyId, i)
        NoticeSend(member, 0, "Party message!")
    end
end
```

---

## Monster & NPC

### `MonsterCreate(monsterid, world, x, y, dir)` → integer
Spawn a monster. Returns object index (-1 on failure).

> ⚠️ Parameter order: **monster ID first**, then world/coordinates.

```lua
local idx = MonsterCreate(5, MAP_LORENCIA, 125, 125, 0)
if idx >= 0 then
    LogPrint("Spawned monster index " .. idx)
end
```

### `MonsterCreateEx(monsterid, world, x, y, dir, ...)` → integer
Spawn with extended parameters (14 total — custom stats, elemental attributes).

### `MonsterDelete(monsterIndex)`
Remove a monster.

```lua
MonsterDelete(idx)
```

---

## Items

See [Item Structures](Item-Structures.md) for item table format.

### `ItemGive(aIndex, bagId)`
Give an item from a predefined bag directly to inventory.

### `ItemGiveEx(aIndex, itemCat, itemIndex, itemLevel, count, opt1, opt2, opt3, ancient, harmony, ...)`
Give a specific item with full option control.

```lua
-- Give +15 Kris with Skill + full Excellent
ItemGiveEx(aIndex, 0, 0, 15, 1, 0, 1, 63, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
```

### `ItemDrop(aIndex, map, x, y, bagId)`
Drop an item from a bag onto the ground.

### `ItemDropEx(aIndex, map, x, y, itemCat, itemIndex, itemLevel, opt1, opt2, opt3, ancient, ...)`
Drop a specific item with full options on the ground.

```lua
ItemDropEx(aIndex,
    GetObjectMap(aIndex), GetObjectMapX(aIndex), GetObjectMapY(aIndex),
    13, 14, 0,         -- Jewel of Chaos
    0, 0, 0, 0,        -- no options
    255,               -- durability
    0, 0, 0, 0, 0, 0,  -- sockets
    0, 0, 0, 0, 0)
```

### `CreateItem(itemCat, itemIndex, itemLevel, durability, opt1, opt2, opt3, ancient, harmony, ...)` → itemId
Build an item and return its ID (does not place it in inventory).

### `InsertItem_GremoryCase(aIndex, itemCat, itemIndex, itemLevel, durability, opt1, opt2, opt3, skill, luck, exc, ancient, ...)`
Insert item directly into player's Gremory Case (reward inbox — safe from disconnect/death).

```lua
-- Give a Jewel of Bless to inbox
InsertItem_GremoryCase(aIndex, 13, 13, 0, 255, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
```

### `InsertItem_GremoryCaseEx(...)`
Extended version with additional parameters.

### `IsItem(cat, idx)` → boolean
Check if item ID is valid.

### `IsSocketItem(itemTable)` → boolean
Check for socket slots.

### `IsElementalItem(itemTable)` → boolean
Check for elemental options.

### `IsPentagramItem(itemTable)` → boolean
Check if pentagram item.

### `Is28Option(itemTable)` → boolean
Check for 2.8 additional option.

### `GetItemKindA(cat, idx)` → integer
Get the item kind type (weapon class / armor type).

### `GetBagItemLevel(slot)` → integer
Get level of a bag item.

### `GetAncientOpt(slot)` → integer
Get ancient option byte.

### `GCTotalFreeSlotCount(aIndex, caseType)` → integer
Free slot count in Gremory Case.

---

## Inventory

See [Item Structures](Item-Structures.md) for the full inventory API.

```lua
-- Quick reference
InventoryGetWearSize(aIndex)
InventoryGetMainSize(aIndex)
InventoryGetFullSize(aIndex)
InventoryGetItemTable(aIndex, slot)          -- item table or nil
InventoryGetItemCount(aIndex)
InventoryGetItemIndex(aIndex, cat, idx)      -- first slot (-1 if none)
InventoryGetFreeSlotCount(aIndex)
InventoryCheckSpaceByItem(aIndex, cat, idx)  -- boolean
InventoryCheckSpaceBySize(aIndex, w, h)      -- boolean
InventorySetItemTable(aIndex, slot, tbl)
InventoryDelItemIndex(aIndex, cat, idx)
InventoryDelItemCount(aIndex, cat, idx, n)

-- Event inventory (same API, different prefix)
EventInventoryGet/Set/Del...

-- Muun inventory
MuunInventoryGet/Set/Del...
```

---

## Player Options Table

### `UserGetOptionTable(aIndex)` → table
Returns a table of all bonus stats active on the player:

```lua
local opts = UserGetOptionTable(aIndex)
local bonus_damage  = opts["AddDamage"]
local bonus_defense = opts["AddDefense"]
local bonus_speed   = opts["AddSpeed"]
local max_hp_bonus  = opts["MaxHP"]
-- also: MaxMP, MaxBP, MaxSD, Recovery*, CriticalDamage, MulDamage, MulDefense, etc.
```

### `UserSetOptionTable(aIndex, table)`
Apply custom option bonuses to a player.

```lua
UserSetOptionTable(aIndex, {
    AddDamage  = 500,
    AddDefense = 200,
    MaxHP      = 10000,
})
UserCalcAttribute(aIndex)
```

---

## Quests & Misc

### `QuestStateCheck(aIndex, questId)` → integer
Get the current state of an evolution quest.

```lua
local state = QuestStateCheck(aIndex, 1)
if state == 1 then
    -- quest complete
end
```

### `RandomGetNumber(max)` → integer
Random integer from `0` to `max` (inclusive).

```lua
local roll = RandomGetNumber(100)
if roll <= 30 then
    -- 30% chance (0–30 out of 0–100)
end
```

### `GetRandomValue(range)` → integer
Random integer from `0` to `range - 1`.

```lua
local idx = GetRandomValue(#rewardTable) + 1  -- random table index
```

---

## Config Files

### `ReadConfigNumber(file, section, key, default)` → number
Read a value from an INI file (relative to Data folder).

```lua
local rate   = ReadConfigNumber("Config/MyPlugin.ini", "Rates", "ExpRate", 1)
local enable = ReadConfigNumber("Config/MyPlugin.ini", "General", "Switch", 1)
```

### `ConfigReadNumber(key, default)` → number
Read from the server's main config.

### `ConfigReadString(key, default)` → string
Read a string from the server's main config.

### `ConfigSaveString(key, value)`
Write a string to the server's main config.

---

## Database (Async SQL)

### `SQLAsyncQuery(label, sql [, callbackParam])`
Execute a non-blocking SQL query. Result arrives in `OnSQLAsyncResult`.

```lua
local name = GetObjectName(aIndex):gsub("'", "''")
SQLAsyncQuery("load_" .. aIndex,
    string.format("SELECT points FROM myplugin WHERE char_name = '%s'", name))
```

See [Database Structures](Database-Structures.md) for full guide.

---

## Practical Examples

### Give a reward safely
```lua
local function GiveReward(aIndex, cat, idx, level)
    if InventoryCheckSpaceByItem(aIndex, cat, idx) then
        ItemGiveEx(aIndex, cat, idx, level, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
    else
        InsertItem_GremoryCase(aIndex, cat, idx, level, 255, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
        NoticeSend(aIndex, 0, "Inventory full — item sent to inbox.")
    end
end
```

### Iterate all online players
```lua
local function ForEachPlayer(fn)
    for i = GetMinUserIndex(), GetMaxUserIndex() do
        if GetObjectConnected(i) == OBJECT_ONLINE then
            fn(i)
        end
    end
end

ForEachPlayer(function(i)
    NoticeSend(i, 1, "Server event starting!")
end)
```

### Apply a timed buff via command
```lua
BridgeFunctionAttach("OnCommandManager", function(aIndex, Type, code, arg)
    if code ~= 50 then return 0 end  -- /buff command

    if EffectCheck(aIndex, 100) then
        NoticeSend(aIndex, 0, "Buff already active.")
        return 1
    end

    EffectAdd(aIndex, 0, 100, 300, 20, 0, 0, 0)  -- 5 minutes
    NoticeSend(aIndex, 0, "Buff applied for 5 minutes!")
    return 1
end)
```

### Mute a player temporarily
```lua
local function MutePlayer(aIndex, seconds)
    PermissionRemove(aIndex, 11)              -- disable chat
    SetObjectChatLimitTime(aIndex, seconds)   -- engine-side timer
    NoticeSend(aIndex, 3, "You have been muted for " .. seconds .. " seconds.")
end
```
