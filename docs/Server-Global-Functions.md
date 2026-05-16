# Server Global Functions

All functions are available as globals — no `require` needed.

---

## Chat & Messaging

```lua
NoticeSend(aIndex, type, message)       -- notice to one player (type: 0=normal 1=blue 2=guild 3=red)
NoticeSendToAll(type, message)          -- broadcast to all
NoticeGlobalSend(type, message)         -- all GS instances
ChatTargetSend(aIndex, npcIndex, msg)   -- NPC dialogue
MessageSend(aIndex, title, body)        -- popup dialog
MessageSendToAll(title, body)           -- popup to all
SendRgbChat(aIndex, r, g, b, message)  -- colored chat
LogPrint(message)                       -- server log
LogColor(color, message)               -- colored console log
```

---

## Commands

```lua
CommandCheckGameMasterLevel(aIndex, level)  -- boolean: GM level check
CommandGetArgNumber(n)                       -- nth arg as number
CommandGetArgString(n)                       -- nth arg as string
CommandSend(aIndex, command)                 -- execute command as player
```

Example:
```lua
-- /addcoins 500
local amount = CommandGetArgNumber(1)  -- 500
```

---

## Effects & Visuals

```lua
EffectAdd(aIndex, effectId, duration, value, casterIndex)
EffectDel(aIndex, effectId)
EffectCheck(aIndex, effectId)   -- boolean
EffectClear(aIndex)
FireworksSend(aIndex, map, x, y, type)
SkinChangeSend(aIndex, skinId)
UserActionSend(aIndex, actionId)
```

---

## Map & World

```lua
MapCheckAttr(map, x, y, attr)  -- boolean: 0=walkable 1=safe 2=no-pk
MapGetItemTable(map, x, y)     -- items on ground
GetMapName(mapId)              -- "Lorencia" etc.
```

---

## Teleport

```lua
MoveUser(aIndex, map, x, y)
MoveUserEx(aIndex, map, x, y, flag)
```

---

## Party

```lua
PartyCreate(aIndex)                      -- create, return partyId
PartyDelete(partyId)
PartyAddMember(partyId, aIndex)
PartyDelMember(partyId, aIndex)
PartyGetMemberCount(partyId)             -- integer
PartyGetMemberIndex(partyId, slot)       -- aIndex at slot (0-based)
```

---

## Monster & NPC

```lua
MonsterCreate(map, x, y, monsterId)           -- returns object index
MonsterCreateEx(map, x, y, monsterId, flag)
MonsterDelete(monsterIndex)
```

---

## Experience & Level

```lua
ExpSend(aIndex, amount)
LevelUpSend(aIndex)
MasterLevelUpSend(aIndex)
MasterSkillTreeRebuild(aIndex)
RuudSend(aIndex, amount)
```

---

## Config Files

```lua
ReadConfigNumber(file, section, key, default)  -- read from INI file
ConfigReadNumber(key, default)
ConfigReadString(key, default)
ConfigSaveString(key, value)
```

---

## Random

```lua
RandomGetNumber(min, max)   -- integer in [min, max]
GetRandomValue(max)         -- integer in [0, max-1]
```

---

## Database

```lua
SQLAsyncQuery(label, sql)
-- result → OnSQLAsyncResult(label, rows)
```

See [Database Structures](Database-Structures.md).

---

## Gremory Case

```lua
InsertItem_GremoryCase(aIndex, itemCat, itemIndex, itemLevel,
    dur, opt1, opt2, opt3, skill, luck, exc, ancient)

InsertItem_GremoryCaseEx(aIndex, itemCat, itemIndex, itemLevel, ...)
```

---

## Quests & Misc

```lua
QuestStateCheck(aIndex, questId)    -- integer: quest state
SetWindowText(text)                 -- server window title
PostSend(aIndex, data)              -- raw packet to client
```

---

## Server Info

```lua
GetGameServerCode()
GetGameServerVersion()
GetGameServerProtocol()
GetGameServerCurUser()     -- online player count
GetGameServerMaxUser()
GetMaxIndex()
GetMinUserIndex() / GetMaxUserIndex()
GetMinMonsterIndex() / GetMaxMonsterIndex()
```
