# Item Structures

> ⚠️ **Read this first — the #1 source of plugin crashes.**
> The engine identifies an item by **one combined number**, never by a separate
> `(category, index)` pair:
>
> ```lua
> local itemid    = itemCat * 512 + itemIndex   -- combine  (== MakeItemID in ItemBagScript.lua)
> local itemCat   = math.floor(itemid / 512)    -- decode category
> local itemIndex = itemid % 512                -- decode index
> ```
>
> **Every** give / drop / check / space function below takes that **combined `itemid`** — *not*
> two args. Wrong argument count or the old `(cat, idx)` split is exactly what makes
> `InsertItem_GremoryCase` / `ItemGiveEx` raise a Lua error and abort your handler. The argument
> counts here are taken straight from `Game/LuaFunction.cpp`; the complete authoritative list
> (all 244 functions, with enforced arg counts) is in
> **[Server Lua Functions](Server-Lua-Functions.md)**.

Inventory slots: **0–11** = equipment (wear), **12+** = main bag. (Slots `236`/`237`/`238` are
special equip areas — pets/wings/etc. — and are also accepted by the slot-based functions.)

---

## Item Categories

There is **no fixed category map** — categories and indices are defined **per server** in
`Data/Item/ItemList.xml`. **Always look the item up there**; do not assume a season's numbering.

Verified examples from this server's `ItemList.xml` (category **14**, "Wings/Orbs/Spheres"):

| Item | `itemCat` | `itemIndex` | `itemid` (`cat*512+idx`) |
|---|---|---|---|
| Jewel of Bless | 14 | 13 | 7181 |
| Jewel of Soul | 14 | 14 | 7182 |
| Jewel of Life | 14 | 16 | 7184 |

> 💡 **Stackable items** (jewels, potions, etc. — see `Data/Item/ItemStack.xml`): the stack
> **count is stored in the durability field**, with `level = 0`. To give 30 Jewels of Soul you
> create **one** item with `durability = 30`, not 30 separate items.

---

## Reading an item — `InventoryGetItemTable(aIndex, slot)`

Returns a table of the item at `slot`, or **nil** if the slot is empty. Fields (exact keys the
engine emits):

| Field | Meaning |
|---|---|
| `Serial` | Unique item serial |
| `Index` | **Combined `itemid`** (`cat*512+idx`) — decode with `/512`, `%512` |
| `Level` | Item level 0–15 |
| `Durability` | Durability (**= stack count** for stackables) |
| `Option1` | Skill flag |
| `Option2` | Luck flag |
| `Option3` | Excellent-options bitmask |
| `NewOption` | Harmony / "new" option (Exe) |
| `SetOption` | Ancient set option |
| `JoHOption` | Jewel-of-Harmony option (currently always 0) |
| `380Option` | +380 option |
| `SocketOption1`…`SocketOption5` | Socket slot values (`255` = empty) |
| `SocketOptionBonus` | Socket bonus option |
| `DurationTime` | Remaining duration (timed items) |
| `ExpireDate` | Expiry timestamp |
| `MasteryExe` | Mastery excellent option |

```lua
local item = InventoryGetItemTable(aIndex, 12)   -- 2 args: aIndex, slot
if item then
    local cat, idx = math.floor(item.Index / 512), item.Index % 512
    LogPrint(string.format("Cat=%d Idx=%d Lv=%d Dur=%d", cat, idx, item.Level, item.Durability))
end
```

---

## Writing an item — `InventorySetItemTable(aIndex, slot, tbl)`

Modifies the item **already present** in `slot` (it does **not** create a new item, and the
item **type/`Index` cannot be changed** here — use `ItemGiveEx` to create). Returns nil if the
slot is empty. Honored keys: `Serial`, `Level`, `Durability`, `Option1`, `Option2`, `Option3`,
`NewOption`, `SetOption`, `380Option`, `SocketOption1`–`SocketOption5`, `SocketOptionBonus`,
`DurationTime`, `ExpireDate` (other keys are ignored).

```lua
-- e.g. max out and bless an existing item in slot 12:
InventorySetItemTable(aIndex, 12, { Level = 15, Durability = 255, Option3 = 0 })
```

---

## Inventory query / delete

```lua
InventoryGetItemTable(aIndex, slot)             -- 2 args -> item table, or nil if empty
InventoryGetItemIndex(aIndex, slot)             -- 2 args -> item Index at slot, nil if empty
InventoryGetItemCount(aIndex, itemid, level)    -- 3 args -> count of matching items
InventoryGetFreeSlotCount(aIndex)               -- 1 arg  -> number of free bag slots
InventoryCheckSpaceByItem(aIndex, itemid)       -- 2 args -> free slot (>=0), or <0 if no room
InventoryCheckSpaceBySize(aIndex, width, height)-- 3 args -> free slot (>=0), or <0 if no room
InventoryDelItemIndex(aIndex, slot)             -- 2 args -> delete the item at slot
InventoryDelItemCount(aIndex, itemid, level, count) -- 4 args -> delete N matching items
```

> ⚠️ `InventoryCheckSpaceByItem` takes the **combined `itemid`** and returns a **slot number**
> (`< 0` means "no room"), **not** a boolean and **not** `(cat, idx)`.

The **Event** inventory mirrors this API with an `EventInventory` prefix
(`EventInventoryGetItemTable` / `GetItemIndex` / `GetItemCount` / `DelItemIndex` / `DelItemCount`),
and the **Muun** inventory with a `MuunInventory` prefix — see
[Server Lua Functions](Server-Lua-Functions.md) for the full set and exact arg counts.

---

## Give / drop items

```lua
-- Create a fully-specified item directly into the player's inventory (>= 8 args).
--   level 0 + durability = stack count  ->  one stacked item.
ItemGiveEx(aIndex, itemid, level, durability, opt1, opt2, opt3, newOption)
-- optional trailing args (in order): setOption, johOption, opt380,
--   socket1, socket2, socket3, socket4, socket5, socketBonus, duration, extraExe

-- Drop a fully-specified item on the ground at map coords (>= 11 args).
ItemDropEx(aIndex, map, x, y, itemid, level, durability, skill, luck, option, newOption)

-- Run a configured ItemBag (Data/Item/ItemBag...) — NOT a direct cat/idx give:
ItemGive(aIndex, itemBagId)              -- 2 args: roll the bag into the inventory
ItemDrop(aIndex, map, x, y, itemBagId)   -- 5 args: roll the bag as a ground drop
```

> ⚠️ `ItemGive` / `ItemDrop` take an **ItemBag id**, not an item. To hand over a *specific*
> item use `ItemGiveEx` (inventory) or `ItemDropEx` (ground).

**Give 30 Jewels of Soul (stackable) into the bag:**

```lua
local itemid = 14 * 512 + 14            -- Jewel of Soul
ItemGiveEx(aIndex, itemid, 0, 30, 0, 0, 0, 0)   -- level 0, durability(=count) 30
```

---

## Gremory Case (reward inbox)

A safe reward that survives disconnect/death. **Exactly 24 arguments**, in this order:

```lua
InsertItem_GremoryCase(
    aIndex,          -- 1  player index
    caseType,        -- 2  0=Account 1=Character 2=Mobile 3=PersonalStore
    giveType,        -- 3  give-type tag (0 is fine)
    itemid,          -- 4  combined cat*512+idx
    level,           -- 5
    dur,             -- 6  durability (= stack count for stackables)
    skill,           -- 7
    luck,            -- 8
    option,          -- 9
    ancient,         -- 10
    socket,          -- 11
    elemental,       -- 12
    muunEvo,         -- 13
    exc,             -- 14
    masteryExc,      -- 15
    socket1, socket2, socket3, socket4, socket5,  -- 16-20
    socketBonus,     -- 21
    receiptDur,      -- 22
    duration,        -- 23
    count            -- 24  number of stack entries (1 for a single stacked item)
)
```

> ⚠️ The old 12-argument `(aIndex, cat, idx, level, dur, …)` form **does not exist** and raises
> a Lua error. There is also `InsertItem_GremoryCaseEx` (**22 args**, with a `380Option` instead
> of the socket/elemental block) — see [Server Lua Functions](Server-Lua-Functions.md).

---

## Safe give pattern (bag → inbox fallback)

```lua
local itemid = 14 * 512 + 13          -- Jewel of Bless
local qty    = 5
local slot = InventoryCheckSpaceByItem(aIndex, itemid)   -- free slot, or <0 if full
if slot and slot >= 0 then
    ItemGiveEx(aIndex, itemid, 0, qty, 0, 0, 0, 0)        -- stacked into the bag
else
    NoticeSend(aIndex, 0, "Inventory full — sent to Gremory Case.")
    -- caseType 1 = Character; one stacked entry (dur = qty, count = 1)
    InsertItem_GremoryCase(aIndex, 1, 0, itemid, 0, qty, 0,0,0,0,0,0,0,0,0, 0,0,0,0,0, 0,0,0, 1)
end
```

---

## Item checks

```lua
IsItem(itemid)            -- 1 arg  -> true if a template exists for this combined id
IsSocketItem(itemid)      -- 1 arg  -> true if a socket-kind item
IsElementalItem(itemid)   -- 1 arg  -> true if pentagram/errtel
IsPentagramItem(itemid)   -- 1 arg  -> true if pentagram
GetItemKindA(itemid)      -- 1 arg  -> the template's Kind1, nil if unknown
Is28Option()              -- 0 args -> stub, always false
```

> ⚠️ These take the **combined `itemid`**, not `(cat, idx)` and not an item table.

---

See **[Server Lua Functions](Server-Lua-Functions.md)** for every engine function with its exact
argument count, and **[Writing Plugins](Writing-Plugins.md)** for the full plugin workflow.
