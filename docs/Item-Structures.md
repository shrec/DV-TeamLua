# Item Structures

Items are identified by `itemCat` (category) and `itemIndex` (index within category).

```lua
local itemId = itemCat * 512 + itemIndex   -- flat ID
local itemCat   = math.floor(itemId / 512) -- decode
local itemIndex = itemId % 512
```

Inventory slots: **0–11** = equipment (wear), **12+** = bag.

---

## Item Categories

| `itemCat` | Category |
|---|---|
| 0–5 | Weapons (swords, axes, maces, spears, bows, staffs) |
| 6 | Shields |
| 7–11 | Armor (helmet, armor, pants, gloves, boots) |
| 12 | Wings |
| 13 | Misc (jewels, potions…) |
| 14 | Orbs |
| 15 | Fenrir / mounts |

---

## Item Table Fields

`InventoryGetItemTable(aIndex, slot)` returns:

| Field | Type | Description |
|---|---|---|
| `ItemCat` | int | Category |
| `ItemIndex` | int | Index in category |
| `Level` | int | Item level 0–15 |
| `Serial` | string | Unique 8-byte serial |
| `Dur` | int | Durability |
| `Option1` | int | Luck / opt byte 1 |
| `Option2` | int | Skill / opt byte 2 |
| `Option3` | int | Excellent options bitmask |
| `AncientOpt` | int | Ancient option |
| `NewOption` | int | Harmony option |
| `SetOption` | int | Socket set byte |
| `JewelOfHarmony` | int | JoH applied |

```lua
local item = InventoryGetItemTable(aIndex, 12)
if item and item.ItemCat >= 0 then
    LogPrint(string.format("Cat=%d Idx=%d Lv=%d", item.ItemCat, item.ItemIndex, item.Level))
end
```

---

## Inventory Query

```lua
InventoryGetWearSize(aIndex)                   -- equip slot count (12)
InventoryGetMainSize(aIndex)                   -- bag size
InventoryGetFullSize(aIndex)                   -- total slots
InventoryGetItemTable(aIndex, slot)            -- item at slot (nil if empty)
InventoryGetItemCount(aIndex)                  -- total items
InventoryGetItemIndex(aIndex, cat, idx)        -- first slot with item (-1 if none)
InventoryGetFreeSlotCount(aIndex)              -- free slots
InventoryCheckSpaceByItem(aIndex, cat, idx)    -- boolean: space for this item?
InventoryCheckSpaceBySize(aIndex, w, h)        -- boolean: space for w×h item?
```

---

## Inventory Modify

```lua
-- Set item at slot
local newItem = { ItemCat=13, ItemIndex=14, Level=0, Dur=255,
                  Option1=0, Option2=0, Option3=0,
                  AncientOpt=0, NewOption=0, SetOption=0, JewelOfHarmony=0 }
InventorySetItemTable(aIndex, 12, newItem)

-- Delete items
InventoryDelItemIndex(aIndex, cat, idx)          -- delete first match
InventoryDelItemCount(aIndex, cat, idx, count)   -- delete N items
```

---

## Give / Drop Items

```lua
-- Give basic item
ItemGive(aIndex, itemCat, itemIndex, itemLevel, count)

-- Give with full options
ItemGiveEx(aIndex, itemCat, itemIndex, itemLevel, count,
           opt1, opt2, opt3, ancient, harmony)

-- Drop on ground
ItemDrop(map, x, y, itemCat, itemIndex, itemLevel, ownerIndex)
ItemDropEx(map, x, y, itemCat, itemIndex, itemLevel,
           opt1, opt2, opt3, ancient, ownerIndex)
```

---

## Gremory Case (Reward Inbox)

Safe reward — survives disconnect/death:
```lua
InsertItem_GremoryCase(aIndex, itemCat, itemIndex, itemLevel,
    dur, opt1, opt2, opt3, skill, luck, exc, ancient)
```

---

## Safe Give Pattern

```lua
if not InventoryCheckSpaceByItem(aIndex, 13, 14) then
    NoticeSend(aIndex, 0, "Inventory full — sent to inbox.")
    InsertItem_GremoryCase(aIndex, 13, 14, 0, 255, 0, 0, 0, 0, 0, 0, 0)
    return
end
ItemGive(aIndex, 13, 14, 0, 1)
```

---

## Event Inventory

Same API, prefixed `EventInventory`:
```lua
EventInventoryGetFullSize / GetItemTable / GetItemIndex / GetItemCount
EventInventorySetItemTable / DelItemIndex / DelItemCount
```

---

## Muun Inventory

```lua
MuunInventoryGetWearSize / GetFullSize / GetItemIndex / GetItemCount
MuunInventoryGetItemTable / SetItemTable / DelItemIndex / DelItemCount
```

---

## Item Checks

```lua
IsItem(cat, idx)             -- valid item?
IsSocketItem(itemTable)      -- has sockets?
IsElementalItem(itemTable)   -- elemental?
IsPentagramItem(itemTable)   -- pentagram?
Is28Option(itemTable)        -- 2.8 option?
GetItemKindA(cat, idx)       -- item kind type
```
