# Player Structure

All functions take `aIndex` as first parameter. Always guard:
```lua
if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
```

---

## Identity

| Function | Returns | Description |
|---|---|---|
| `GetObjectConnected(aIndex)` | int | `0`=offline `1`=connected `2`=logged `3`=online |
| `GetObjectType(aIndex)` | int | `1`=user `2`=monster `3`=npc |
| `GetObjectName(aIndex)` | string | Character name |
| `GetObjectAccount(aIndex)` | string | Account name |
| `GetObjectAccountID(aIndex)` | int | Account DB ID |
| `GetObjectIpAddress(aIndex)` | string | IP address |
| `GetObjectHardwareId(aIndex)` | string | Hardware fingerprint |
| `GetObjectPlayerID(aIndex)` | int | Internal player ID |
| `GetObjectIndexByName(name)` | int | Find index by name (-1 if offline) |
| `GetPlayerMac(aIndex)` | string | MAC address |
| `GetPlayerSerial(aIndex)` | string | Hardware serial |

---

## Class & Level

| Function | Returns |
|---|---|
| `GetObjectClass(aIndex)` | Base class constant (CLASS_DW … CLASS_IK) |
| `GetObjectChangeUp(aIndex)` | Class tier: 0=base 1=2nd 2=3rd |
| `GetObjectLevel(aIndex)` | Character level |
| `GetObjectMasterLevel(aIndex)` | Master level |
| `GetObjectMajesticLevel(aIndex)` | Majestic level |
| `GetObjectToTalLevel(aIndex)` | Combined total level |
| `GetObjectLevelUpPoint(aIndex)` | Available stat points |
| `GetObjectMasterPoint(aIndex)` | Available master tree points |
| `GetObjectMajesticPoint(aIndex)` | Available majestic points |
| `GetObjectReset(aIndex)` | Reset count |
| `GetObjectMasterReset(aIndex)` | Master reset count |

---

## Stats

| Group | Functions |
|---|---|
| Base | `GetObjectStrength/Dexterity/Vitality/Energy/Leadership(aIndex)` |
| Extra (items/buffs) | `GetObjectExtra[Stat](aIndex)` |
| Total | `GetObjectTotal[Stat](aIndex)` |
| Default | `GetObjectDefault[Stat](aIndex)` |

---

## HP / Mana / BP / Shield

| Function | Description |
|---|---|
| `GetObjectLive(aIndex)` | `true` if alive |
| `GetObjectLife/MaxLife(aIndex)` | Current / max HP |
| `GetObjectMana/MaxMana(aIndex)` | Current / max mana |
| `GetObjectBP/MaxBP(aIndex)` | Current / max BP |
| `GetObjectShield/MaxShield(aIndex)` | Current / max AG shield |

```lua
local pct = GetObjectLife(aIndex) / GetObjectMaxLife(aIndex) * 100
```

---

## Currency

| Function | Description |
|---|---|
| `GetObjectMoney(aIndex)` | Zen |
| `GetObjectRuud(aIndex)` | Ruud |
| `ObjectGetCoin(aIndex, type)` | Custom coin (type = tier) |
| `CashShopGetPoint(aIndex)` | WCoin balance |
| `CashShopAddPoint(aIndex, amount)` | Add WCoins |
| `CashShopSubPoint(aIndex, amount)` | Subtract WCoins |
| `ObjectAddCoin(aIndex, type, amount)` | Add custom coins |
| `ObjectSubCoin(aIndex, type, amount)` | Subtract custom coins |

---

## Map & Position

| Function | Description |
|---|---|
| `GetObjectMap(aIndex)` | Current map ID |
| `GetObjectMapX/Y(aIndex)` | Current coordinates |
| `GetObjectDeathMap/X/Y(aIndex)` | Death location |

---

## Guild & Party

| Function | Description |
|---|---|
| `GetObjectPartyNumber(aIndex)` | Party ID (-1 if none) |
| `GetObjectGuildNumber(aIndex)` | Guild ID |
| `GetObjectGuildName(aIndex)` | Guild name |
| `GetObjectGuildStatus(aIndex)` | Guild rank |
| `GetObjectGuildUnionName(aIndex)` | Union name |
| `GetObjectCSGuildSide(aIndex)` | Castle Siege side |

---

## PK, Gens, VIP

| Function | Description |
|---|---|
| `GetObjectPKCount/Level/Timer(aIndex)` | PK stats |
| `GetObjectGensFamily/Rank/Symbol/Contribution(aIndex)` | Gens |
| `GetObjectAccountLevel(aIndex)` | VIP level (0=normal, 1-4=VIP, <0=banned) |
| `GetObjectAccountExpireDate(aIndex)` | VIP expiry string |

---

## Setters

| Function | Description |
|---|---|
| `SetObjectLevel(aIndex, v)` | Set level |
| `SetObjectLevelUpPoint(aIndex, v)` | Set stat points |
| `SetObjectMoney(aIndex, v)` | Set Zen |
| `SetObjectRuud(aIndex, v)` | Set Ruud |
| `SetObjectStrength/Dexterity/Vitality/Energy/Leadership(aIndex, v)` | Base stats |
| `SetObjectMasterLevel/Point(aIndex, v)` | Master |
| `SetObjectMajesticLevel/Point(aIndex, v)` | Majestic |
| `SetObjectLife/MaxLife/Mana/MaxMana/BP/MaxBP/Shield/MaxShield(aIndex, v)` | Resources |
| `SetObjectPKCount/Level/Timer(aIndex, v)` | PK |
| `SetObjectReset(aIndex, v)` | Resets |
| `SetObjectMap/X/Y(aIndex, v)` | Position |
| `SetObjectChatLimitTime(aIndex, secs)` | Mute |
| `ChangeObjectReset(aIndex)` | Full reset process |

After bulk stat changes:
```lua
SetObjectStrength(aIndex, GetObjectStrength(aIndex) + 100)
UserCalcAttribute(aIndex)
LevelUpSend(aIndex)
```

---

## Actions & Sync

| Function | Description |
|---|---|
| `UserCalcAttribute(aIndex)` | Recalculate all derived stats |
| `UserInfoSend(aIndex)` | Resend character info to client |
| `LevelUpSend(aIndex)` | Level-up effect + sync |
| `MasterLevelUpSend(aIndex)` | Master level-up effect |
| `MasterSkillTreeRebuild(aIndex)` | Rebuild master skill tree |
| `ExpSend(aIndex, amount)` | Give EXP |
| `RuudSend(aIndex, amount)` | Give Ruud + sync |
| `SkinChangeSend(aIndex, skinId)` | Visual skin override |
| `UserActionSend(aIndex, actionId)` | Play animation |
| `UserDisconnect(aIndex)` | Force disconnect |
| `UserGameLogout(aIndex)` | Graceful logout |
| `UserWarehouseOpen(aIndex)` | Open warehouse UI |
| `UserSetAccountLevel(aIndex, level)` | Set VIP level |
| `PKLevelSend(aIndex)` | Sync PK level |
| `CalculateCharacter(aIndex)` | Full recalculation |
| `ResetMajesticTree(aIndex)` | Reset majestic tree |
| `UpdatePlayerLevels(aIndex)` | Sync all level data |

---

## Permissions

```lua
PermissionCheck(aIndex, permId)   -- boolean
PermissionInsert(aIndex, permId)  -- grant
PermissionRemove(aIndex, permId)  -- revoke
```
