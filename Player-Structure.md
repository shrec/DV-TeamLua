# Player Structure

All functions accept a player **index** (`aIndex`) as their first argument. Always validate the player is online before reading properties.

```lua
if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
```

---

## Connection & Identity

| Function | Returns | Description |
|---|---|---|
| `GetObjectConnected(aIndex)` | integer | Connection state: `OBJECT_OFFLINE/CONNECTED/LOGGED/ONLINE` |
| `GetObjectType(aIndex)` | integer | `OBJECT_USER`, `OBJECT_MONSTER`, etc. |
| `GetObjectIpAddress(aIndex)` | string | Player's IP address |
| `GetObjectHardwareId(aIndex)` | string | Hardware fingerprint |
| `GetObjectAccount(aIndex)` | string | Account name |
| `GetObjectAccountID(aIndex)` | integer | Account database ID |
| `GetObjectName(aIndex)` | string | Character name |
| `GetObjectPersonalCode(aIndex)` | string | Personal identification code |
| `GetObjectPlayerID(aIndex)` | integer | Internal player numeric ID |
| `GetObjectIndexByName(name)` | integer | Look up index by character name (-1 if offline) |
| `GetPlayerMac(aIndex)` | string | MAC address |
| `GetPlayerSerial(aIndex)` | string | Hardware serial |

```lua
local name    = GetObjectName(aIndex)
local account = GetObjectAccount(aIndex)
local ip      = GetObjectIpAddress(aIndex)
```

---

## Class & Level

| Function | Returns | Description |
|---|---|---|
| `GetObjectClass(aIndex)` | integer | Base class (see class constants) |
| `GetObjectChangeUp(aIndex)` | integer | Class tier (0=base, 1=2nd class, 2=3rd class) |
| `GetObjectLevel(aIndex)` | integer | Character level |
| `GetObjectMasterLevel(aIndex)` | integer | Master level |
| `GetObjectMajesticLevel(aIndex)` | integer | Majestic level |
| `GetObjectToTalLevel(aIndex)` | integer | Total combined level |
| `GetObjectLevelUpPoint(aIndex)` | integer | Available stat points |
| `GetObjectMasterPoint(aIndex)` | integer | Available master tree points |
| `GetObjectMajesticPoint(aIndex)` | integer | Available majestic tree points |
| `GetObjectReset(aIndex)` | integer | Reset count |
| `GetObjectMasterReset(aIndex)` | integer | Master reset count |

```lua
local level  = GetObjectLevel(aIndex)
local class  = GetObjectClass(aIndex)
local resets = GetObjectReset(aIndex)

if class == CLASS_DK and level >= 400 then
    -- Dark Knight with level 400+
end
```

---

## Stats (Base / Extra / Total)

Stats come in three layers: base (allocated points), extra (bonus from items/skills), and total.

| Group | Functions |
|---|---|
| **Base** | `GetObjectStrength`, `GetObjectDexterity`, `GetObjectVitality`, `GetObjectEnergy`, `GetObjectLeadership` |
| **Extra** | `GetObjectExtraStrength`, `GetObjectExtraDexterity`, `GetObjectExtraVitality`, `GetObjectExtraEnergy`, `GetObjectExtraLeadership` |
| **Total** | `GetObjectTotalStrength`, `GetObjectTotalDexterity`, `GetObjectTotalVitality`, `GetObjectTotalEnergy`, `GetObjectTotalLeadership` |
| **Default** | `GetObjectDefaultStrength`, `GetObjectDefaultDexterity`, `GetObjectDefaultVitality`, `GetObjectDefaultEnergy`, `GetObjectDefaultLeadership` |

All functions take `(aIndex)` and return an integer.

```lua
local totalStr = GetObjectTotalStrength(aIndex)
local baseVit  = GetObjectVitality(aIndex)
```

---

## HP / Mana / BP / Shield

| Function | Returns | Description |
|---|---|---|
| `GetObjectLive(aIndex)` | boolean | `true` if alive |
| `GetObjectLife(aIndex)` | integer | Current HP |
| `GetObjectMaxLife(aIndex)` | integer | Maximum HP |
| `GetObjectMana(aIndex)` | integer | Current mana |
| `GetObjectMaxMana(aIndex)` | integer | Maximum mana |
| `GetObjectBP(aIndex)` | integer | Current BP (stamina) |
| `GetObjectMaxBP(aIndex)` | integer | Maximum BP |
| `GetObjectShield(aIndex)` | integer | Current AG shield |
| `GetObjectMaxShield(aIndex)` | integer | Maximum AG shield |

```lua
local hpPct = GetObjectLife(aIndex) / GetObjectMaxLife(aIndex) * 100
if hpPct < 20 then
    NoticeSend(aIndex, 0, "Warning: low HP!")
end
```

---

## Currency

| Function | Returns | Description |
|---|---|---|
| `GetObjectMoney(aIndex)` | integer | Zen (gold) |
| `GetObjectRuud(aIndex)` | integer | Ruud balance |
| `ObjectGetCoin(aIndex, type)` | integer | Custom coin balance (`type` = coin tier) |

```lua
local zen   = GetObjectMoney(aIndex)
local ruud  = GetObjectRuud(aIndex)
local coins = ObjectGetCoin(aIndex, 1)  -- tier-1 coins
```

### Cash Shop

| Function | Returns | Description |
|---|---|---|
| `CashShopGetPoint(aIndex)` | integer | Current WCoin/cash-shop balance |
| `CashShopAddPoint(aIndex, amount)` | — | Add points |
| `CashShopSubPoint(aIndex, amount)` | — | Subtract points |

---

## Map & Position

| Function | Returns | Description |
|---|---|---|
| `GetObjectMap(aIndex)` | integer | Current map ID |
| `GetObjectMapX(aIndex)` | integer | Current X coordinate |
| `GetObjectMapY(aIndex)` | integer | Current Y coordinate |
| `GetObjectDeathMap(aIndex)` | integer | Map where character died |
| `GetObjectDeathMapX(aIndex)` | integer | Death X |
| `GetObjectDeathMapY(aIndex)` | integer | Death Y |

```lua
local map = GetObjectMap(aIndex)
local x   = GetObjectMapX(aIndex)
local y   = GetObjectMapY(aIndex)

if map == MAP_LORENCIA then
    -- player is in Lorencia
end
```

---

## Guild & Party

| Function | Returns | Description |
|---|---|---|
| `GetObjectPartyNumber(aIndex)` | integer | Party ID (-1 if none) |
| `GetObjectGuildNumber(aIndex)` | integer | Guild ID |
| `GetObjectGuildStatus(aIndex)` | integer | Guild rank/status |
| `GetObjectGuildName(aIndex)` | string | Guild name |
| `GetObjectGuildRelationship(aIndex)` | integer | Guild alliance relationship |
| `GetObjectGuildUnionNumber(aIndex)` | integer | Guild union ID |
| `GetObjectGuildUnionName(aIndex)` | string | Guild union name |
| `GetObjectCSGuildSide(aIndex)` | integer | Castle Siege guild side |

---

## PK System

| Function | Returns | Description |
|---|---|---|
| `GetObjectPKCount(aIndex)` | integer | Total PK kill count |
| `GetObjectPKLevel(aIndex)` | integer | PK level (0=white, 1=hero, 2=commoner, 3=outlaw, etc.) |
| `GetObjectPKTimer(aIndex)` | integer | Remaining PK penalty time |

```lua
if GetObjectPKLevel(aIndex) >= 3 then
    NoticeSend(aIndex, 0, "You are an outlaw!")
end
```

---

## Gens

| Function | Returns | Description |
|---|---|---|
| `GetObjectGensFamily(aIndex)` | integer | Gens family (0=none, 1=Duprian, 2=Vant) |
| `GetObjectGensRank(aIndex)` | integer | Gens rank |
| `GetObjectGensSymbol(aIndex)` | integer | Gens symbol count |
| `GetObjectGensContribution(aIndex)` | integer | Contribution points |

---

## Account & VIP

| Function | Returns | Description |
|---|---|---|
| `GetObjectAccountLevel(aIndex)` | integer | Account privilege level (`0`=regular, `1-4`=VIP tiers, negative=banned) |
| `GetObjectAccountExpireDate(aIndex)` | string | VIP expiry date string |
| `GetObjectOfflineFlag(aIndex)` | integer | Offline mode status |

```lua
local vipLevel = GetObjectAccountLevel(aIndex)
if vipLevel >= 1 then
    -- VIP player
end
```

---

## Interface State

| Function | Returns | Description |
|---|---|---|
| `GetObjectInterface(aIndex)` | integer | Current open UI window ID |
| `GetObjectChange(aIndex)` | integer | Character change flag |
| `GetObjectAuthority(aIndex)` | integer | Authority bits |
| `GetObjectLastGainedExp(aIndex)` | integer | EXP from last kill |
| `GetObjectMaxNormalPoints(aIndex)` | integer | Max normal stat points |

---

## Setters

Use setters to modify player properties. Changes take effect immediately in memory; use `UserCalcAttribute(aIndex)` to recalculate derived stats after bulk changes.

| Function | Description |
|---|---|
| `SetObjectLevel(aIndex, value)` | Set character level |
| `SetObjectLevelUpPoint(aIndex, value)` | Set stat points |
| `SetObjectMoney(aIndex, value)` | Set Zen |
| `SetObjectRuud(aIndex, value)` | Set Ruud |
| `SetObjectStrength(aIndex, value)` | Set base STR |
| `SetObjectDexterity(aIndex, value)` | Set base DEX |
| `SetObjectVitality(aIndex, value)` | Set base VIT |
| `SetObjectEnergy(aIndex, value)` | Set base ENE |
| `SetObjectLeadership(aIndex, value)` | Set base CMD |
| `SetObjectMasterLevel(aIndex, value)` | Set master level |
| `SetObjectMasterPoint(aIndex, value)` | Set master points |
| `SetObjectMajesticLevel(aIndex, value)` | Set majestic level |
| `SetObjectMajesticPoint(aIndex, value)` | Set majestic points |
| `SetObjectLife(aIndex, value)` | Set current HP |
| `SetObjectMaxLife(aIndex, value)` | Set max HP |
| `SetObjectMana(aIndex, value)` | Set current mana |
| `SetObjectMaxMana(aIndex, value)` | Set max mana |
| `SetObjectBP(aIndex, value)` | Set current BP |
| `SetObjectMaxBP(aIndex, value)` | Set max BP |
| `SetObjectShield(aIndex, value)` | Set AG shield |
| `SetObjectMaxShield(aIndex, value)` | Set max AG shield |
| `SetObjectPKCount(aIndex, value)` | Set PK count |
| `SetObjectPKLevel(aIndex, value)` | Set PK level |
| `SetObjectPKTimer(aIndex, value)` | Set PK timer |
| `SetObjectLive(aIndex, value)` | Set alive status |
| `SetObjectReset(aIndex, value)` | Set reset count |
| `SetObjectMap(aIndex, map)` | Warp to map |
| `SetObjectMapX(aIndex, x)` | Set X position |
| `SetObjectMapY(aIndex, y)` | Set Y position |
| `SetObjectChatLimitTime(aIndex, seconds)` | Mute for N seconds |
| `ChangeObjectReset(aIndex)` | Trigger full reset process |

```lua
-- Give stats and recalculate
SetObjectStrength(aIndex, GetObjectStrength(aIndex) + 100)
UserCalcAttribute(aIndex)
LevelUpSend(aIndex)
```

---

## Player Actions

| Function | Description |
|---|---|
| `UserCalcAttribute(aIndex)` | Recalculate all derived stats |
| `UserInfoSend(aIndex)` | Resend character info packet to client |
| `LevelUpSend(aIndex)` | Trigger level-up visual/sound effect |
| `MasterLevelUpSend(aIndex)` | Trigger master level-up effect |
| `MasterSkillTreeRebuild(aIndex)` | Rebuild master skill tree |
| `ExpSend(aIndex, amount)` | Give experience points |
| `SkinChangeSend(aIndex, skinId)` | Apply a visual skin override |
| `UserActionSend(aIndex, actionId)` | Play a character animation |
| `UserDisconnect(aIndex)` | Force disconnect |
| `UserGameLogout(aIndex)` | Graceful game logout |
| `UserWarehouseOpen(aIndex)` | Open the warehouse UI |
| `UserSetAccountLevel(aIndex, level)` | Set VIP level |
| `PKLevelSend(aIndex)` | Sync PK level to client |
| `UpdatePlayerLevels(aIndex)` | Sync all level data to client |
| `CalculateCharacter(aIndex)` | Full server-side character recalculation |
| `ResetMajesticTree(aIndex)` | Reset majestic tree |

---

## Permissions

| Function | Returns | Description |
|---|---|---|
| `PermissionCheck(aIndex, permId)` | boolean | Check if player has permission |
| `PermissionInsert(aIndex, permId)` | — | Grant permission |
| `PermissionRemove(aIndex, permId)` | — | Revoke permission |

---

## Custom Coins

| Function | Returns | Description |
|---|---|---|
| `ObjectGetCoin(aIndex, type)` | integer | Get coin balance |
| `ObjectAddCoin(aIndex, type, amount)` | — | Add coins |
| `ObjectSubCoin(aIndex, type, amount)` | — | Subtract coins |

---

## Server Info

| Function | Returns | Description |
|---|---|---|
| `GetMaxIndex()` | integer | Maximum object index |
| `GetMinUserIndex()` | integer | Minimum player index |
| `GetMaxUserIndex()` | integer | Maximum player index |
| `GetMinMonsterIndex()` | integer | Minimum monster/NPC index |
| `GetMaxMonsterIndex()` | integer | Maximum monster/NPC index |
| `GetGameServerCode()` | integer | This server's code/ID |
| `GetGameServerVersion()` | integer | Server version number |
| `GetGameServerProtocol()` | integer | Protocol version |
| `GetGameServerCurUser()` | integer | Current online player count |
| `GetGameServerMaxUser()` | integer | Server player cap |
| `GetGameServerLang()` | integer | Server language code |
| `GetObjectLang(aIndex)` | integer | Player's client language |
