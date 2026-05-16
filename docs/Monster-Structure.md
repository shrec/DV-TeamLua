# Monster Structure

Monsters and NPCs share the same index system as players.

---

## Identifying Monsters

```lua
if GetObjectType(bIndex) == OBJECT_MONSTER then end
if GetObjectType(bIndex) == OBJECT_NPC     then end
```

---

## Reading Properties

```lua
GetObjectType(bIndex)       -- OBJECT_MONSTER or OBJECT_NPC
GetObjectMap(bIndex)        -- map ID
GetObjectMapX(bIndex)       -- X coordinate
GetObjectMapY(bIndex)       -- Y coordinate
GetObjectLevel(bIndex)      -- monster level
GetObjectLife(bIndex)       -- current HP
GetObjectMaxLife(bIndex)    -- max HP
GetObjectLive(bIndex)       -- boolean: alive?
```

---

## Spawn / Remove

```lua
-- Spawn
local idx = MonsterCreate(map, x, y, monsterId)
local idx = MonsterCreateEx(map, x, y, monsterId, flag)
--   flag: 0=normal  1=no auto-respawn  2=event-linked

-- Remove
MonsterDelete(idx)
```

---

## Kill Event

```lua
BridgeFunctionAttach("OnMonsterDie", function(aIndex, bIndex, skill)
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
    if GetObjectType(bIndex) ~= OBJECT_MONSTER then return end

    local map = GetObjectMap(bIndex)
    local x   = GetObjectMapX(bIndex)
    local y   = GetObjectMapY(bIndex)

    -- Drop item at kill location
    ItemDrop(map, x, y, 13, 14, 0, aIndex)
end)
```

---

## Kill Counter

```lua
local kills = {}

BridgeFunctionAttach("OnMonsterDie", function(aIndex, bIndex, skill)
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
    if GetObjectType(bIndex) ~= OBJECT_MONSTER then return end

    local name = GetObjectName(aIndex)
    kills[name] = (kills[name] or 0) + 1

    if kills[name] % 100 == 0 then
        NoticeSend(aIndex, 0, kills[name] .. " monsters killed!")
        InsertItem_GremoryCase(aIndex, 13, 14, 0, 255, 0, 0, 0, 0, 0, 0, 0)
    end
end)

BridgeFunctionAttach("OnCharacterClose", function(aIndex)
    kills[GetObjectName(aIndex)] = nil
end)
```

---

## Iterate All Monsters on a Map

```lua
local function ForEachMonster(targetMap, fn)
    for i = GetMinMonsterIndex(), GetMaxMonsterIndex() do
        if GetObjectType(i) == OBJECT_MONSTER
        and GetObjectLive(i)
        and GetObjectMap(i) == targetMap then
            fn(i)
        end
    end
end

-- Delete all monsters in Lorencia
ForEachMonster(MAP_LORENCIA, MonsterDelete)
```

---

## Custom NPC Talk

```lua
BridgeFunctionAttach("OnNpcTalk", function(aIndex, npcIndex)
    if GetObjectMap(npcIndex) == MAP_LORENCIA
    and GetObjectMapX(npcIndex) == 125
    and GetObjectMapY(npcIndex) == 125 then
        ChatTargetSend(aIndex, npcIndex, "Hello! I am a custom NPC.")
        return 1  -- suppress default menu
    end
    return 0
end)
```
