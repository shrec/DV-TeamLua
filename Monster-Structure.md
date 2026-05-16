# Monster Structure

Monsters and NPCs share the same object system as players. They are referenced by an integer index and queried with the same `GetObject*` functions.

---

## Identifying Monsters

Use `GetObjectType()` to distinguish monsters from players:

```lua
BridgeFunctionAttach("OnMonsterDie", function(aIndex, bIndex, skill)
    if GetObjectType(bIndex) ~= OBJECT_MONSTER then return end
    -- bIndex is definitely a monster
end)
```

Object type constants:
```lua
OBJECT_NONE    = 0
OBJECT_USER    = 1
OBJECT_MONSTER = 2
OBJECT_NPC     = 3
OBJECT_ITEM    = 4
```

---

## Reading Monster Properties

These functions work on monster indices exactly like player indices:

| Function | Returns | Description |
|---|---|---|
| `GetObjectType(bIndex)` | integer | `OBJECT_MONSTER` or `OBJECT_NPC` |
| `GetObjectMap(bIndex)` | integer | Map the monster is on |
| `GetObjectMapX(bIndex)` | integer | Current X coordinate |
| `GetObjectMapY(bIndex)` | integer | Current Y coordinate |
| `GetObjectLevel(bIndex)` | integer | Monster level |
| `GetObjectLife(bIndex)` | integer | Current HP |
| `GetObjectMaxLife(bIndex)` | integer | Maximum HP |
| `GetObjectLive(bIndex)` | boolean | `true` if alive |

---

## Spawning Monsters

### `MonsterCreate(map, x, y, monsterId)` → integer
Spawn a monster and return its object index. Returns `-1` on failure.

```lua
local idx = MonsterCreate(MAP_LORENCIA, 125, 125, 5)
if idx >= 0 then
    LogPrint("Spawned monster at index " .. idx)
end
```

### `MonsterCreateEx(map, x, y, monsterId, flag)` → integer
Spawn with extra behavior flags.

| `flag` | Effect |
|---|---|
| `0` | Normal |
| `1` | No automatic respawn |
| `2` | Event-linked |

```lua
-- Spawn a boss that won't auto-respawn
local boss = MonsterCreateEx(MAP_LORENCIA, 125, 125, 78, 1)
```

---

## Removing Monsters

### `MonsterDelete(monsterIndex)`
Remove a monster from the world immediately.

```lua
MonsterDelete(bossIndex)
```

---

## Monster Kill Events

### `OnMonsterDie(aIndex, bIndex, skill)`

| Parameter | Type | Description |
|---|---|---|
| `aIndex` | integer | Killer (player) index |
| `bIndex` | integer | Monster index |
| `skill` | integer | Skill ID of killing blow (0 = normal attack) |

```lua
BridgeFunctionAttach("OnMonsterDie", function(aIndex, bIndex, skill)
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
    if GetObjectType(bIndex) ~= OBJECT_MONSTER then return end

    local map    = GetObjectMap(bIndex)
    local x      = GetObjectMapX(bIndex)
    local y      = GetObjectMapY(bIndex)
    local killer = GetObjectName(aIndex)

    -- Example: drop a custom item when boss (id >= 78) dies
    -- (monster ID check would need additional engine support)
    ItemDrop(map, x, y, 13, 14, 0, aIndex)  -- drop Jewel of Chaos
end)
```

---

## Kill Counter Plugin Example

Track how many monsters each player kills and reward at milestones:

```lua
local killCounts = {}

BridgeFunctionAttach("OnReadScript", function()
    killCounts = {}
end)

BridgeFunctionAttach("OnMonsterDie", function(aIndex, bIndex, skill)
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
    if GetObjectType(bIndex) ~= OBJECT_MONSTER then return end

    local name = GetObjectName(aIndex)
    killCounts[name] = (killCounts[name] or 0) + 1
    local count = killCounts[name]

    -- Reward every 100 kills
    if count % 100 == 0 then
        NoticeSend(aIndex, 0, string.format("Kill milestone: %d monsters!", count))
        InsertItem_GremoryCase(aIndex, 13, 14, 0, 255, 0, 0, 0, 0, 0, 0, 0)
    end
end)

BridgeFunctionAttach("OnCharacterClose", function(aIndex)
    killCounts[GetObjectName(aIndex)] = nil
end)
```

---

## Map Iteration

To apply logic to all monsters on a map, iterate from `GetMinMonsterIndex()` to `GetMaxMonsterIndex()`:

```lua
local function ForEachMonsterOnMap(targetMap, callback)
    local minIdx = GetMinMonsterIndex()
    local maxIdx = GetMaxMonsterIndex()

    for i = minIdx, maxIdx do
        if GetObjectType(i) == OBJECT_MONSTER
        and GetObjectLive(i)
        and GetObjectMap(i) == targetMap then
            callback(i)
        end
    end
end

-- Usage: delete all monsters in Lorencia
ForEachMonsterOnMap(MAP_LORENCIA, function(idx)
    MonsterDelete(idx)
end)
```

---

## NPC Talk

React to a player clicking on an NPC:

```lua
BridgeFunctionAttach("OnNpcTalk", function(aIndex, npcIndex)
    if GetObjectType(npcIndex) ~= OBJECT_NPC then return 0 end

    -- Custom handler for a specific NPC location
    local map = GetObjectMap(npcIndex)
    local x   = GetObjectMapX(npcIndex)
    local y   = GetObjectMapY(npcIndex)

    if map == MAP_LORENCIA and x == 125 and y == 125 then
        ChatTargetSend(aIndex, npcIndex, "Hello! I am a custom NPC.")
        return 1  -- handled, suppress default NPC menu
    end

    return 0  -- let the default menu open
end)
```
