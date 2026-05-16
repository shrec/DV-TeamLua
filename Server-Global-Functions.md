# Server Global Functions

Complete reference for all C++ functions available in server-side Lua scripts. Functions are available as globals — no `require` needed.

For player property getters/setters see [Player Structure](Player-Structure).
For inventory/item functions see [Item Structures](Item-Structures).

---

## Chat & Messaging

### `NoticeSend(aIndex, type, message)`
Send an in-game notice to one player.

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Target player index |
| `type` | integer | Notice type: `0`=normal, `1`=blue, `2`=guild, `3`=red |
| `message` | string | Message text |

```lua
NoticeSend(aIndex, 0, "Server restart in 5 minutes.")
```

---

### `NoticeSendToAll(type, message)`
Broadcast a notice to every online player.

```lua
NoticeSendToAll(0, "Event starting now!")
```

---

### `NoticeGlobalSend(type, message)`
Global server-wide notice (shown on all connected game servers in multi-GS setup).

```lua
NoticeGlobalSend(1, "Castle Siege begins!")
```

---

### `ChatTargetSend(aIndex, targetIndex, message)`
Send a chat message appearing to come from another object (NPC dialogue).

```lua
ChatTargetSend(aIndex, npcIndex, "Greetings, adventurer!")
```

---

### `MessageSend(aIndex, title, body)`
Send a popup dialog with title and body text.

```lua
MessageSend(aIndex, "Daily Reward", "You received 100 coins!")
```

---

### `MessageSendToAll(title, body)`
Send a popup dialog to all online players.

---

### `MessageGet(aIndex)` → string
Read the last message sent by the player (from chat input).

---

### `SendRgbChat(aIndex, r, g, b, message)`
Send a colored chat message.

```lua
SendRgbChat(aIndex, 255, 215, 0, "You found a rare item!")
```

---

### `LogPrint(message)`
Write a message to the server log (console + log file).

```lua
LogPrint("[MyPlugin] Player " .. GetObjectName(aIndex) .. " used command")
```

---

### `LogColor(color, message)`
Write a colored log message (console only).

```lua
LogColor(12, "[ERROR] something went wrong")  -- 12 = red
```

---

## Commands

### `CommandCheckGameMasterLevel(aIndex, level)` → boolean
Returns `true` if the player's GM level meets or exceeds `level`.

```lua
if not CommandCheckGameMasterLevel(aIndex, 1) then
    NoticeSend(aIndex, 0, "No permission.")
    return 1
end
```

---

### `CommandGetArgNumber(n)` → integer
Get the Nth argument of the current command as a number (1-indexed).

```lua
-- /addcoins 500
local amount = CommandGetArgNumber(1)  -- returns 500
```

---

### `CommandGetArgString(n)` → string
Get the Nth argument of the current command as a string.

```lua
-- /kick PlayerName
local targetName = CommandGetArgString(1)
```

---

### `CommandSend(aIndex, command)`
Execute a command on behalf of a player.

---

## Effects & Visuals

### `EffectAdd(aIndex, effectId, duration, value, casterIndex)`
Apply a buff/effect to a player or monster.

```lua
EffectAdd(aIndex, 100, 30, 0, aIndex)  -- effect 100, 30 seconds
```

---

### `EffectDel(aIndex, effectId)`
Remove a specific effect.

---

### `EffectCheck(aIndex, effectId)` → boolean
Check if a player currently has an effect.

```lua
if EffectCheck(aIndex, 100) then
    -- player has the effect
end
```

---

### `EffectClear(aIndex)`
Remove all effects from a player.

---

### `FireworksSend(aIndex, map, x, y, type)`
Trigger a fireworks/visual particle at a location.

```lua
FireworksSend(aIndex, GetObjectMap(aIndex), GetObjectMapX(aIndex), GetObjectMapY(aIndex), 1)
```

---

### `SkinChangeSend(aIndex, skinId)`
Override a player's visual skin.

---

### `UserActionSend(aIndex, actionId)`
Force a character to play an animation.

---

## Map & World

### `MapCheckAttr(map, x, y, attr)` → boolean
Check a tile attribute at coordinates.

| Attr | Meaning |
|---|---|
| `0` | Is tile walkable? |
| `1` | Safe zone |
| `2` | No PK zone |

```lua
if MapCheckAttr(GetObjectMap(aIndex), x, y, 1) then
    -- target tile is in safe zone
end
```

---

### `MapGetItemTable(map, x, y)` → table
Get items on the ground at a map tile.

---

### `GetMapName(mapId)` → string
Returns the name of a map by ID.

```lua
local name = GetMapName(MAP_LORENCIA)  -- "Lorencia"
```

---

## Party

### `PartyCreate(aIndex)` → integer
Create a party with `aIndex` as leader. Returns party ID.

### `PartyDelete(partyId)`
Disband a party.

### `PartyAddMember(partyId, aIndex)` → boolean
Add a player to an existing party.

### `PartyDelMember(partyId, aIndex)`
Remove a player from a party.

### `PartyGetMemberCount(partyId)` → integer
Get current member count.

### `PartyGetMemberIndex(partyId, slot)` → integer
Get the player index at a party slot (0-based).

```lua
local count = PartyGetMemberCount(partyId)
for i = 0, count - 1 do
    local memberIndex = PartyGetMemberIndex(partyId, i)
    NoticeSend(memberIndex, 0, "Party message!")
end
```

---

## Monster & NPC

### `MonsterCreate(map, x, y, monsterId)` → integer
Spawn a monster. Returns the object index.

```lua
local idx = MonsterCreate(MAP_LORENCIA, 100, 100, 5)
```

---

### `MonsterCreateEx(map, x, y, monsterId, flag)` → integer
Spawn with extra flags (e.g. custom AI).

---

### `MonsterDelete(monsterIndex)`
Remove a monster from the world.

---

## Teleport

### `MoveUser(aIndex, map, x, y)`
Teleport a player instantly.

```lua
MoveUser(aIndex, MAP_DEVIAS, 215, 40)
```

---

### `MoveUserEx(aIndex, map, x, y, flag)`
Teleport with extra options.

---

## Database (Async SQL)

### `SQLAsyncQuery(label, sql)`
Execute a non-blocking SQL query. The result is delivered via `OnSQLAsyncResult`.

```lua
local name = GetObjectName(aIndex):gsub("'", "''")
SQLAsyncQuery("myplug_load",
    string.format("SELECT points FROM myplug WHERE char_name = '%s'", name))
```

Result in callback:
```lua
BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if label ~= "myplug_load" then return end
    local pts = rows[1] and rows[1]["points"] or 0
end)
```

See [Database Structures](Database-Structures) for full guide.

---

## Config Files

### `ReadConfigNumber(file, section, key, default)` → number
Read a numeric value from an INI-style config file.

```lua
local rate = ReadConfigNumber("Config.ini", "Rates", "ExpRate", 1)
```

---

### `ConfigReadNumber(key, default)` → number
Read from the server's default config.

### `ConfigReadString(key, default)` → string
Read a string value from the server's default config.

### `ConfigSaveString(key, value)`
Write a value to the server config.

---

## Random

### `RandomGetNumber(min, max)` → integer
Return a random integer in `[min, max]`.

```lua
local roll = RandomGetNumber(1, 100)
if roll <= 30 then
    -- 30% chance
end
```

### `GetRandomValue(max)` → integer
Return a random integer in `[0, max-1]`.

---

## Quests

### `QuestStateCheck(aIndex, questId)` → integer
Return the current state of a quest for a player.

---

## Experience

### `ExpSend(aIndex, amount)`
Give experience to a player and update the client.

```lua
ExpSend(aIndex, 1000000)
```

### `LevelUpSend(aIndex)`
Trigger the level-up animation and sync to client.

### `MasterLevelUpSend(aIndex)`
Trigger master level-up animation.

### `MasterSkillTreeRebuild(aIndex)`
Force a rebuild of the master skill tree UI.

---

## Window / UI

### `SetWindowText(text)`
Set the server process window title (visible in task manager / console).

```lua
SetWindowText("DevEmu GS1 | Online: " .. GetGameServerCurUser())
```

---

## Utility

### `GetObjectConnected(aIndex)` → integer
Connection state. One of:

```lua
OBJECT_OFFLINE   = 0
OBJECT_CONNECTED = 1
OBJECT_LOGGED    = 2
OBJECT_ONLINE    = 3
```

### `GetObjectType(aIndex)` → integer
Object type. One of:

```lua
OBJECT_NONE    = 0
OBJECT_USER    = 1
OBJECT_MONSTER = 2
OBJECT_NPC     = 3
OBJECT_ITEM    = 4
```

### `PostSend(aIndex, data)`
Send a raw post/packet to a player's client.

---

## Ruud Currency

### `RuudSend(aIndex, amount)`
Give Ruud to a player and sync to client.

```lua
RuudSend(aIndex, 500)
```

---

## Gremory Case

### `InsertItem_GremoryCase(aIndex, itemCat, itemIndex, itemLevel, durability, option1, option2, option3, skill, luck, excellent, ancient)`
Insert an item directly into the player's Gremory Case (reward inbox).

```lua
InsertItem_GremoryCase(aIndex, 13, 0, 0, 255, 0, 0, 0, 0, 0, 0, 0)
```

### `InsertItem_GremoryCaseEx(aIndex, itemCat, itemIndex, itemLevel, ...)`
Extended version with additional option fields.
