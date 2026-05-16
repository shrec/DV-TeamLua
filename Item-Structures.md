# Item Structures

Items are referenced by category and index: `itemId = itemCat * 512 + itemIndex`.

Inventory slots: **0–11** = equipment slots (wear), **12+** = bag slots.

---

## Item ID Formula

```lua
-- Converting from itemCat + itemIndex to a flat ID
local itemId = itemCat * 512 + itemIndex

-- Decoding a flat ID
local itemCat   = math.floor(itemId / 512)
local itemIndex = itemId % 512
```

### Item Categories

| `itemCat` | Category |
|---|---|
| 0 | Swords / short weapons |
| 1 | Axes |
| 2 | Maces / clubs |
| 3 | Spears |
| 4 | Bows |
| 5 | Staffs |
| 6 | Shields |
| 7 | Helmets |
| 8 | Armors |
| 9 | Pants |
| 10 | Gloves |
| 11 | Boots |
| 12 | Wings |
| 13 | Misc (jewels, potions, sets…) |
| 14 | Orbs / special |
| 15 | Fenrir / mounts |

---

## Item Table Structure

`InventoryGetItemTable(aIndex, slot)` returns a table with these fields:

| Field | Type | Description |
|---|---|---|
| `ItemCat` | integer | Item category |
| `ItemIndex` | integer | Item index within category |
| `Level` | integer | Item level (0–15) |
| `Serial` | string | Unique 8-byte serial (hex string) |
| `Dur` | integer | Durability |
| `Option1` | integer | Luck / option byte 1 |
| `Option2` | integer | Skill / option byte 2 |
| `Option3` | integer | Excellent options bitmask |
| `AncientOpt` | integer | Ancient option byte |
| `NewOption` | integer | Harmony / additional option |
| `SetOption` | integer | Socket option set byte |
| `JewelOfHarmony` | integer | JoH applied |

```lua
local item = InventoryGetItemTable(aIndex, 12)  -- slot 12 (first bag slot)
if item and item.ItemCat >= 0 then
    LogPrint(string.format("Cat=%d Idx=%d Lv=%d", item.ItemCat, item.ItemIndex, item.Level))
end
```

---

## Inventory Query

### `InventoryGetWearSize(aIndex)` → integer
Number of equipment slots (usually 12).

### `InventoryGetMainSize(aIndex)` → integer
Size of the main bag section.

### `InventoryGetFullSize(aIndex)` → integer
Total inventory slot count (wear + bag).

### `InventoryGetItemTable(aIndex, slot)` → table | nil
Get item at slot. Returns `nil` if slot is empty.

### `InventoryGetItemCount(aIndex)` → integer
Total items in inventory.

### `InventoryGetItemIndex(aIndex, itemCat, itemIndex)` → integer
Find the first slot containing a specific item type. Returns `-1` if not found.

```lua
local slot = InventoryGetItemIndex(aIndex, 13, 14)  -- find Jewel of Chaos
if slot >= 0 then
    -- player has it
end
```

### `InventoryGetFreeSlotCount(aIndex)` → integer
Number of free inventory slots.

### `InventoryCheckSpaceByItem(aIndex, itemCat, itemIndex)` → boolean
Check if there is space to add a specific item (accounts for item grid size).

### `InventoryCheckSpaceBySize(aIndex, width, height)` → boolean
Check if there is space for an item of given dimensions.

---

## Inventory Modification

### `InventorySetItemTable(aIndex, slot, itemTable)`
Write an item table into a slot. The `itemTable` must have at minimum `ItemCat`, `ItemIndex`, and `Level`.

```lua
local newItem = {
    ItemCat   = 13,
    ItemIndex = 14,   -- Jewel of Chaos
    Level     = 0,
    Dur       = 255,
    Option1   = 0, Option2 = 0, Option3 = 0,
    AncientOpt = 0, NewOption = 0, SetOption = 0,
    JewelOfHarmony = 0,
}
InventorySetItemTable(aIndex, 12, newItem)
```

### `InventoryDelItemIndex(aIndex, itemCat, itemIndex)`
Delete the first occurrence of an item type from inventory.

### `InventoryDelItemCount(aIndex, itemCat, itemIndex, count)`
Delete `count` items of a given type.

```lua
-- Remove 3 Jewels of Chaos
InventoryDelItemCount(aIndex, 13, 14, 3)
```

---

## Event Inventory

Same API as regular inventory, prefixed with `EventInventory`:

```lua
EventInventoryGetFullSize(aIndex)
EventInventoryGetItemTable(aIndex, slot)
EventInventoryGetItemIndex(aIndex, itemCat, itemIndex)
EventInventoryGetItemCount(aIndex)
EventInventorySetItemTable(aIndex, slot, itemTable)
EventInventoryDelItemIndex(aIndex, itemCat, itemIndex)
EventInventoryDelItemCount(aIndex, itemCat, itemIndex, count)
```

---

## Giving Items

### `ItemGive(aIndex, itemCat, itemIndex, itemLevel, count)`
Give a basic item directly into inventory.

```lua
ItemGive(aIndex, 13, 14, 0, 1)  -- give 1 Jewel of Chaos
```

### `ItemGiveEx(aIndex, itemCat, itemIndex, itemLevel, count, option1, option2, option3, ancient, harmony)`
Give an item with full option specification.

```lua
ItemGiveEx(aIndex, 0, 0, 15, 1, 0, 1, 63, 0, 0)  -- +15 Kris with luck, skill, full excellent
```

---

## Dropping Items (Ground)

### `ItemDrop(map, x, y, itemCat, itemIndex, itemLevel, ownerIndex)`
Spawn an item on the ground at map coordinates.

```lua
ItemDrop(GetObjectMap(aIndex), GetObjectMapX(aIndex), GetObjectMapY(aIndex), 13, 14, 0, aIndex)
```

### `ItemDropEx(map, x, y, itemCat, itemIndex, itemLevel, option1, option2, option3, ancient, ownerIndex)`
Drop item with full options.

---

## Creating Items (Internal)

### `CreateItem(itemCat, itemIndex, itemLevel, option1, option2, option3, ancient)` → table
Build an item table without placing it anywhere. Useful for bag inserts.

---

## Bag Items

### `AddItemBag(aIndex, itemTable)`
Add an item to the player's bag (overflow/special bag inventory).

### `InsertItem_GremoryCase(aIndex, itemCat, itemIndex, itemLevel, dur, opt1, opt2, opt3, skill, luck, exc, ancient)`
Insert directly into the Gremory Case (reward inbox — safe from death/disconnect).

```lua
InsertItem_GremoryCase(aIndex, 13, 0, 0, 255, 0, 0, 0, 0, 0, 0, 0)
-- gives a Box of Luck to the player's inbox
```

---

## Item Checks

### `IsItem(itemCat, itemIndex)` → boolean
Check if an item ID is valid.

### `IsSocketItem(itemTable)` → boolean
Check if an item has socket slots.

### `IsElementalItem(itemTable)` → boolean
Check if an item has elemental properties.

### `IsPentagramItem(itemTable)` → boolean
Check if an item is a Pentagram item.

### `Is28Option(itemTable)` → boolean
Check if an item has a 2.8 additional option.

### `GetItemKindA(itemCat, itemIndex)` → integer
Get the item kind type A (weapon class / armor type).

### `GetBagItemLevel(slot)` → integer
Get the level of a bag item.

### `GetAncientOpt(slot)` → integer
Get the ancient option byte of a bag item.

---

## Muun Inventory

| Function | Description |
|---|---|
| `MuunInventoryGetWearSize(aIndex)` | Muun equipment slots |
| `MuunInventoryGetFullSize(aIndex)` | Total Muun inventory size |
| `MuunInventoryGetItemIndex(aIndex, cat, idx)` | Find Muun item slot |
| `MuunInventoryGetItemCount(aIndex)` | Muun item count |
| `MuunInventoryGetItemTable(aIndex, slot)` | Get Muun item at slot |
| `MuunInventorySetItemTable(aIndex, slot, tbl)` | Set Muun item at slot |
| `MuunInventoryDelItemIndex(aIndex, cat, idx)` | Delete first matching Muun item |
| `MuunInventoryDelItemCount(aIndex, cat, idx, n)` | Delete N matching Muun items |

---

## Practical Examples

### Check if player has 3 Jewels of Bless then take them
```lua
local slot = InventoryGetItemIndex(aIndex, 13, 13)  -- Jewel of Bless
if slot < 0 then
    NoticeSend(aIndex, 0, "You need 3 Jewels of Bless.")
    return 1
end
local count = InventoryGetItemCount(aIndex)
-- (a real implementation would count stacks)
InventoryDelItemCount(aIndex, 13, 13, 3)
NoticeSend(aIndex, 0, "Items consumed!")
```

### Give a reward item safely
```lua
if not InventoryCheckSpaceByItem(aIndex, 12, 0) then
    NoticeSend(aIndex, 0, "No inventory space! Item sent to Gremory Case.")
    InsertItem_GremoryCase(aIndex, 12, 0, 0, 255, 0, 0, 0, 0, 0, 0, 0)
    return
end
ItemGive(aIndex, 12, 0, 0, 1)
```
