# Server Lua Functions

Complete reference of the **server-side Lua engine functions** the GameServer exposes (`Game/LuaFunction.cpp`, registered in `InitLuaFunction`). A plugin's `server.lua` can call any of these by name — they resolve through the plugin's sandboxed `_ENV` to the trusted `_G`. This table is generated from the C++ argument-extraction code, so the **argument counts and types are authoritative** (not guessed from prose).

> ⚠️ **Argument count is enforced.** Most functions check `lua_gettop(L)` and raise a Lua error if the count is wrong — e.g. `InsertItem_GremoryCase` requires **exactly 24** args, `ItemGiveEx` needs **≥ 8**, `InventoryCheckSpaceByItem` needs **exactly 2**. Match the count and order exactly; wrong arity is the most common plugin crash (it aborts the calling `OnInvoke`).
>
> 💡 **Item ids are a single number:** `itemid = type * 512 + index` (the `MakeItemID(type, index)` helper in `ItemBagScript.lua`). `ItemGiveEx` / `ItemGive` / `InsertItem_GremoryCase` / `InventoryCheckSpaceByItem` take the **combined `itemid`**, never separate `(category, index)` args. For a **stackable** item the stack count is the **durability** field (level 0).
>
> 💡 **`aIndex`** is the player/object in-world index (from `ctx:aIndex()` in a plugin). A `?` on a parameter marks it optional; `N+` in Args means N required plus optional trailing args.

See also [Writing Plugins](Writing-Plugins.md) and [MUPF Server API](MUPF-Server-API.md).

| Function | Args | Parameters | Returns | Purpose |
|---|---|---|---|---|
| `GetMaxIndex()` | 0 | none | int max object index (g_MaxObjects) | Returns the max object array index |
| `GetGameServerLang()` | 0 | none | int default language | Returns the server's default language |
| `GetObjectLang(aIndex)` | 1 | aIndex:int | int (always 0) | Returns object language (stub, always 0) |
| `GetMinUserIndex()` | 0 | none | int (g_StartPlayers) | Returns the lowest player object index |
| `GetMaxUserIndex()` | 0 | none | int (g_EndPlayers) | Returns the highest player object index |
| `GetMinMonsterIndex()` | 0 | none | int (g_StartMonsters) | Returns the lowest monster object index |
| `GetMaxMonsterIndex()` | 0 | none | int (g_EndMonsters) | Returns the highest monster object index |
| `GetGameServerCode()` | 0 | none | int server code | Returns the configured server code |
| `GetGameServerVersion()` | 0 | none | int version | Returns the game server version |
| `GetGameServerProtocol()` | 0 | none | int (always 0) | Returns server protocol (stub, always 0) |
| `GetGameServerCurUser()` | 0 | none | int current user count | Returns count of online characters |
| `GetGameServerMaxUser()` | 0 | none | int max users (MaxPlayersSet) | Returns the configured max player capacity |
| `GetObjectConnected(aIndex)` | 1 | aIndex:int | int status (0 offline, 3 playing) | Returns a player's connection status |
| `GetObjectIpAddress(aIndex)` | 1 | aIndex:int | string IP | Returns a player's IP address |
| `GetObjectHardwareId(aIndex)` | 1 | aIndex:int | string (stub, null) | Returns a player's hardware id (stub) |
| `GetObjectType(aIndex)` | 1 | aIndex:int | int type (USER/NPC/ITEM/MONSTER) | Returns the object's type category |
| `GetObjectAccount(aIndex)` | 1 | aIndex:int | string account | Returns a player's account name |
| `GetObjectAccountID(aIndex)` | 1 | aIndex:int | int account GUID | Returns a player's account DB GUID |
| `GetObjectPassword(aIndex)` | 1 | aIndex:int | string (empty) | Returns player password (stub, empty) |
| `GetObjectName(aIndex)` | 1 | aIndex:int | string name | Returns monster name or player DB name |
| `GetObjectPersonalCode(aIndex)` | 1 | aIndex:int | string (empty) | Returns personal code (stub, empty) |
| `GetObjectClass(aIndex)` | 1 | aIndex:int | int class | Returns the object's class id |
| `GetObjectChangeUp(aIndex)` | 1 | aIndex:int | int evolution level | Returns a player's evolution (change-up) level |
| `GetObjectLevel(aIndex)` | 1 | aIndex:int | int level | Returns player normal level or monster level |
| `GetObjectMasterLevel(aIndex)` | 1 | aIndex:int | int master level | Returns a player's master level |
| `GetObjectMajesticLevel(aIndex)` | 1 | aIndex:int | int majestic level | Returns a player's majestic level |
| `GetObjectToTalLevel(aIndex)` | 1 | aIndex:int | int total level | Returns a player's combined total level |
| `GetObjectLevelUpPoint(aIndex)` | 1 | aIndex:int | int free stat points | Returns a player's unspent level-up points |
| `GetObjectRuud(aIndex)` | 1 | aIndex:int | int ruud | Returns a player's ruud money |
| `GetObjectMoney(aIndex)` | 1 | aIndex:int | int zen | Returns a player's zen money |
| `GetObjectStrength(aIndex)` | 1 | aIndex:int | int strength | Returns a player's strength stat |
| `GetObjectDexterity(aIndex)` | 1 | aIndex:int | int agility | Returns a player's agility/dexterity stat |
| `GetObjectVitality(aIndex)` | 1 | aIndex:int | int vitality | Returns a player's vitality stat |
| `GetObjectEnergy(aIndex)` | 1 | aIndex:int | int energy | Returns a player's energy stat |
| `GetObjectLeadership(aIndex)` | 1 | aIndex:int | int leadership | Returns a player's leadership stat |
| `GetObjectExtraStrength(aIndex)` | 1 | aIndex:int | int added strength | Returns a player's bonus strength (from items/buffs) |
| `GetObjectExtraDexterity(aIndex)` | 1 | aIndex:int | int added agility | Returns a player's bonus agility |
| `GetObjectExtraVitality(aIndex)` | 1 | aIndex:int | int added vitality | Returns a player's bonus vitality |
| `GetObjectExtraEnergy(aIndex)` | 1 | aIndex:int | int added energy | Returns a player's bonus energy |
| `GetObjectExtraLeadership(aIndex)` | 1 | aIndex:int | int added leadership | Returns a player's bonus leadership |
| `GetObjectDefaultStrength(aIndex)` | 1 | aIndex:int | int base strength | Returns the class base strength for a player |
| `GetObjectDefaultDexterity(aIndex)` | 1 | aIndex:int | int base agility | Returns the class base agility for a player |
| `GetObjectDefaultVitality(aIndex)` | 1 | aIndex:int | int base vitality | Returns the class base vitality for a player |
| `GetObjectDefaultEnergy(aIndex)` | 1 | aIndex:int | int base energy | Returns the class base energy for a player |
| `GetObjectDefaultLeadership(aIndex)` | 1 | aIndex:int | int base leadership | Returns the class base leadership for a player |
| `GetObjectLive(aIndex)` | 1 | aIndex:int | int alive flag | Returns whether the object is alive |
| `GetObjectLife(aIndex)` | 1 | aIndex:int | int current HP | Returns the object's current life/HP |
| `GetObjectMaxLife(aIndex)` | 1 | aIndex:int | int max HP | Returns the object's maximum life/HP |
| `GetObjectMana(aIndex)` | 1 | aIndex:int | int current mana | Returns the object's current mana |
| `GetObjectMaxMana(aIndex)` | 1 | aIndex:int | int max mana | Returns the object's maximum mana |
| `GetObjectBP(aIndex)` | 1 | aIndex:int | int current stamina/BP | Returns the object's current ability/BP (stamina) |
| `GetObjectMaxBP(aIndex)` | 1 | aIndex:int | int max stamina/BP | Returns the object's maximum ability/BP |
| `GetObjectShield(aIndex)` | 1 | aIndex:int | int current shield | Returns the object's current shield (SD) |
| `GetObjectMaxShield(aIndex)` | 1 | aIndex:int | int (max shield power) | Returns object's max shield value |
| `GetObjectPKCount(aIndex)` | 1 | aIndex:int | int | Returns player's PK kill count |
| `GetObjectPKLevel(aIndex)` | 1 | aIndex:int | int | Returns player's PK level |
| `GetObjectPKTimer(aIndex)` | 1 | aIndex:int | int (always 0) | Returns player PK timer (stubbed to 0) |
| `GetObjectMap(aIndex)` | 1 | aIndex:int | int | Returns object's current world/map id |
| `GetObjectMapX(aIndex)` | 1 | aIndex:int | int | Returns object's map X coordinate |
| `GetObjectMapY(aIndex)` | 1 | aIndex:int | int | Returns object's map Y coordinate |
| `GetObjectDeathMap(aIndex)` | 1 | aIndex:int | int | Returns object's last-location world id |
| `GetObjectDeathMapX(aIndex)` | 1 | aIndex:int | int | Returns object's last-location X coordinate |
| `GetObjectDeathMapY(aIndex)` | 1 | aIndex:int | int | Returns object's last-location Y coordinate |
| `GetObjectAuthority(aIndex)` | 1 | aIndex:int | int | Returns player's command authority level |
| `GetObjectPartyNumber(aIndex)` | 1 | aIndex:int | int | Returns player's party id |
| `GetObjectGuildNumber(aIndex)` | 1 | aIndex:int | int | Returns player's guild id |
| `GetObjectGuildStatus(aIndex)` | 1 | aIndex:int | int (rank, 255 if none) | Returns player's guild member ranking |
| `GetObjectGuildName(aIndex)` | 1 | aIndex:int | string | Returns player's guild db name ("" if none) |
| `GetObjectGuildRelationship(aIndex, bIndex)` | 2 | aIndex:int, bIndex:int | int (1 if same guild else 0) | Tests if target player is a guild member of source |
| `GetObjectGuildUnionNumber(aIndex)` | 1 | aIndex:int | int | Returns player's guild alliance id |
| `GetObjectGuildUnionName(aIndex)` | 1 | aIndex:int | string | Returns player's guild alliance (union) name |
| `GetObjectChange(aIndex)` | 1 | aIndex:int | int | Returns player's evolution (change up) level |
| `GetObjectInterface(aIndex)` | 1 | aIndex:int | string | Returns player's current interface-state id |
| `GetObjectAccountLevel(aIndex)` | 1 | aIndex:int | int | Returns player's account VIP status |
| `UserSetAccountLevel(aIndex, aValue, bValue)` | 3 | aIndex:int, aValue:int, bValue:int | nil | Sets player VIP status/duration if expiry not past |
| `GetObjectAccountExpireDate(aIndex)` | 1 | aIndex:int | int (VIP duration/expiry) | Returns player's VIP expiry timestamp |
| `GetObjectReset(aIndex)` | 1 | aIndex:int | int | Returns player's reset count |
| `GetObjectMasterReset(aIndex)` | 1 | aIndex:int | int | Returns player's reset count (master reset stub) |
| `GetObjectGensRank(aIndex)` | 1 | aIndex:int | int | Returns player's Gens ranking |
| `GetObjectGensSymbol(aIndex)` | 1 | aIndex:int | int (always 0) | Returns player's Gens symbol (stubbed to 0) |
| `GetObjectGensFamily(aIndex)` | 1 | aIndex:int | int | Returns player's Gens family |
| `GetObjectGensContribution(aIndex)` | 1 | aIndex:int | int | Returns player's Gens contribution points |
| `GetObjectCSGuildSide(aIndex)` | 1 | aIndex:int | int (attacker side flag) | Returns player's castle-siege attacker side |
| `GetObjectOfflineFlag(aIndex)` | 1 | aIndex:int | int (1 if offline-helper else 0) | Returns whether player is in offline-helper mode |
| `GetObjectIndexByName(aString)` | 1 | aString:string | int (entry, -1 if not found) | Resolves an online player index from name |
| `CashShopGetPoint(aIndex)` | 1 | aIndex:int | table (WCoinC, WCoinP, GoblinPoint) | Returns player's cash-shop point balances |
| `CashShopAddPoint(aIndex, aValue, bValue, cValue)` | 4 | aIndex:int, aValue:int, bValue:int, cValue:int | nil | Adds WCoinC credits and goblin points to player |
| `CashShopSubPoint(aIndex, aValue, bValue, cValue)` | 4 | aIndex:int, aValue:int, bValue:int, cValue:int | nil | Subtracts WCoinC credits and goblin points from player |
| `SetObjectLevel(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | Sets monster level or player normal level |
| `SetObjectLive(aIndex)` | 0 | (none) | nil | No-op stub |
| `SetObjectLife(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | Sets unit's current life and resends to player |
| `SetObjectMaxLife(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | Sets unit's max life and resends to player |
| `SetObjectMana(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | Sets unit's current mana and resends to player |
| `SetObjectMaxMana(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | Sets unit's max mana and resends to player |
| `ChangeObjectReset(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | Sets player reset count and saves character |
| `SetObjectBP(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | Sets unit's current stamina/BP and resends |
| `SetObjectMaxBP(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | Sets unit's max stamina/BP and resends |
| `SetObjectShield(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | Sets unit's current shield and resends to player |
| `SetObjectMaxShield(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets a unit's max shield power and resends life |
| `SetObjectReset(...)` | 0 | none | nil | no-op stub, always returns success |
| `SetObjectLevelUpPoint(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets player's normal level-up points (clamped >=0) |
| `SetObjectMoney(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets player zen (clamped 0..2,000,000,000) |
| `SetObjectRuud(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets player Ruud money (clamped 0..2B) and sends update |
| `SetObjectStrength(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets STRENGTH stat (clamped class-min..35000), recalcs and sends |
| `SetObjectDexterity(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets AGILITY stat (clamped class-min..35000), recalcs and sends |
| `SetObjectVitality(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets VITALITY stat (clamped class-min..35000), recalcs and sends |
| `SetObjectEnergy(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets ENERGY stat (clamped class-min..35000), recalcs and sends |
| `SetObjectLeadership(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets LEADERSHIP stat (clamped class-min..35000), recalcs and sends |
| `SetObjectChatLimitTime(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | mutes player chat for given time |
| `SetObjectPKCount(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets player PK count (clamped >=0) |
| `SetObjectPKLevel(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets player PK level (clamped >=0) |
| `SetObjectPKTimer(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | validates player; PK timer assignment commented out (no-op) |
| `SetObjectMap(aIndex, aValue)` | 1 | aIndex:int, aValue:int | nil | sets player's world/map id (guard requires only 1 arg) |
| `SetObjectMapX(aIndex, aValue)` | 1 | aIndex:int, aValue:int | nil | sets player's X coordinate (guard requires only 1 arg) |
| `SetObjectMapY(aIndex, aValue)` | 1 | aIndex:int, aValue:int | nil | sets player's Y coordinate (guard requires only 1 arg) |
| `SetObjectMasterLevel(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets master level (if master), sends status and recalcs |
| `SetObjectMasterPoint(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets master points (if master), sends status and recalcs |
| `SetObjectMajesticLevel(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets majestic level (if majestic), sends status and recalcs |
| `SetObjectMajesticPoint(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets majestic points (if master), sends status and recalcs |
| `ChatTargetSend(aIndex, bIndex, aString)` | 3 | aIndex:int, bIndex:int, aString:string | nil | sends NPC TEXT_SAY chat to one player (bIndex) or all viewport players (bIndex=-1) |
| `CommandCheckGameMasterLevel(aIndex, aValue)` | 2 | aIndex:int, aValue:int | int (1 if master level>=aValue, else 0) | checks whether player's master level meets a threshold |
| `CommandGetArgNumber(aString, aValue)` | 2 | aString:string, aValue:int | int (parsed number) | extracts the Nth space-delimited token from a command as an integer |
| `CommandGetArgString(aString, aValue)` | 2 | aString:string, aValue:int | string (token) | extracts the Nth space-delimited token from a command as a string |
| `CommandSend(...)` | 0 | none | nil | no-op stub (body commented out) |
| `ReadConfigNumber(aString, bString, cString)` / `ConfigReadNumber(...)` | 3 | aString:string(section), bString:string(key), cString:string(file) | int | reads an integer value from an INI config file |
| `ConfigReadString(aString, bString, cString)` | 3 | aString:string(section), bString:string(key), cString:string(file) | string | reads a string value from an INI config file |
| `ConfigSaveString(aString, bString, cString, dString)` | 4 | aString:string(section), bString:string(key), cString:string(value), dString:string(file) | nil | writes a string value to an INI config file |
| `FireworksSend(aIndex, x, y)` | 3 | aIndex:int, x:int, y:int | nil | sends fireworks effect at coords (defaults to player pos if x=y=0) |
| `InventoryGetWearSize()` | 0 | none | int (PLAYER_MAX_EQUIPMENT) | returns equipment slot count |
| `InventoryGetMainSize()` | 0 | none | int (normal_inventory_size) | returns main inventory size |
| `InventoryGetFullSize()` | 0 | none | int (INVENTORY_EXT4_SIZE) | returns full inventory size including extensions |
| `EventInventoryGetFullSize()` | 0 | none | int (EVENT_INVENTORY_SIZE) | returns event inventory size |
| `EventInventoryGetItemTable(aIndex, aValue)` | 2 | aIndex:int, aValue:int(slot) | table (item fields) or nil | returns a table of all item fields at an event-inventory slot |
| `EventInventorySetItemTable(aIndex, Slot, itemTable)` | 3 | aIndex:int, Slot:int, itemTable:table | nil | writes item fields from a table into an event-inventory slot and resends |
| `InventorySetItemTable(aIndex, Slot, t)` | 3 | aIndex:int, Slot:int, t:table | nil | Apply a Lua item-attribute table onto a main-inventory slot, convert and resend it |
| `InventoryGetItemTable(aIndex, slot)` | 2 | aIndex:int, slot:int | table (item fields) or nil | Read a main-inventory slot's item into a Lua attribute table |
| `EventInventoryGetItemIndex(aIndex, slot)` | 2 | aIndex:int, slot:int | int item index, nil if empty | Get the item-index of an event-inventory slot |
| `InventoryGetItemIndex(aIndex, slot)` | 2 | aIndex:int, slot:int | int item index, nil if empty | Get the item-index of a main-inventory slot |
| `EventInventoryGetItemCount(aIndex, itemid, level)` | 3 | aIndex:int, itemid:int, level:number | int count | Count matching items in the event inventory |
| `InventoryGetItemCount(aIndex, itemid, level)` | 3 | aIndex:int, itemid:int, level:number | int count | Count matching items in the main inventory |
| `EventInventoryDelItemIndex(aIndex, slot)` | 2 | aIndex:int, slot:int | nil | Delete an event-inventory item by slot and resend the event item list |
| `InventoryDelItemIndex(aIndex, slot)` | 2 | aIndex:int, slot:int | nil | Delete a main-inventory item by slot (ItemDeleteByUse) |
| `InventoryDelItemCount(aIndex, itemid, level, count)` | 4 | aIndex:int, itemid:int, level:number, count:int | nil | Delete N matching items from the main inventory |
| `EventInventoryDelItemCount(aIndex, itemid, level, count)` | 4 | aIndex:int, itemid:int, level:number, count:int | nil | Delete N matching items from the event inventory |
| `InventoryGetFreeSlotCount(aIndex)` | 1 | aIndex:int | int empty-slot count | Get number of free main-inventory slots |
| `InventoryCheckSpaceByItem(aIndex, itemid)` | 2 | aIndex:int, itemid:int | int slot, <0 if no space | Find a free inventory position fitting the given item |
| `InventoryCheckSpaceBySize(aIndex, width, height)` | 3 | aIndex:int, width:int, height:int | int slot, <0 if no space | Find a free inventory position for a width×height region |
| `ItemDrop(aIndex, map, x, y, itemBagId)` | 5 | aIndex:int, map:int, x:int, y:int, itemBagId:int | nil | Execute an ItemBag as a ground drop at the given map coords |
| `ItemDropEx(aIndex, map, x, y, itemid, level, durability, skill, luck, option, newopt, setopt?, johOption?, opt380?, socket1?, socket2?, socket3?, socket4?, socket5?, socketBonus?, duration?, lootInd?, extraExe?)` | 11+ | aIndex:int, map:int, x:int, y:int, itemid:int, level:int, durability:int, skill:int, luck:int, option:int, newopt:int, setopt?:int, johOption?:int, opt380?:int, socket1?:int, socket2?:int, socket3?:int, socket4?:int, socket5?:int, socketBonus?:int, duration?:int, lootInd?:int, extraExe?:int | nil | Build a fully-specified item and drop it on the ground (serial-created) |
| `ItemGive(aIndex, itemBagId)` | 2 | aIndex:int, itemBagId:int | nil | Run an ItemBag to insert its rolled item into the player's inventory |
| `GetItemKindA(itemid)` | 1 | itemid:int | int Kind1, nil if unknown item | Return the item template's Kind1 category |
| `ItemGiveEx(aIndex, itemid, level, durability, opt1, opt2, opt3, newOption, setOption?, johOption?, opt380?, socket1?, socket2?, socket3?, socket4?, socket5?, socketBonus?, duration?, extraExe?)` | 8+ | aIndex:int, itemid:int, level:int, durability:int, Option1:int, Option2:int, Option3:int, NewOption:int, SetOption?:int, JoHOption?:int, i380Option?:int, SocketOption1?:int, SocketOption2?:int, SocketOption3?:int, SocketOption4?:int, SocketOption5?:int, SocketOptionBonus?:int, Duration?:int, ExtraExe?:int | nil | Build a fully-specified item and create it directly into the player's inventory |
| `LevelUpSend(aIndex)` | 1 | aIndex:int | nil | Send the normal level-up packet to the player |
| `LogPrint(str)` | 1 | str:string | nil | Write a string to the "lua" info log |
| `LogColor(color, str)` | 2 | color:int (unused), str:string | nil | Write a string to the "lua" info log (color arg ignored) |
| `MapCheckAttr(worldId, x, y, attr)` | 4 | worldId:int, x:int, y:int, attr:int | int (attribute test result, 0 if no world) | Test a terrain attribute bit at map coords |
| `MapGetItemTable(aIndex, itemIndex)` | 2 | aIndex:int, itemIndex:int | table (item fields) or nil | Read a ground item in the player's world into a Lua attribute table |
| `MasterLevelUpSend(aIndex)` | 1 | aIndex:int | nil | Send the master level-up packet (only if player is master) |
| `MasterSkillTreeRebuild(aIndex, group)` | 2 | aIndex:int, group:int | nil | Reset/rebuild a master skill tree group |
| `MessageGet(msgId)` | 1 | msgId:int | string message text | Look up a localized message string by id |
| `MoneySend(aIndex, amount)` | 2 | aIndex:int, amount:int | nil | Add (or subtract) zen to the player, clamp 0..2,000,000,000, and resend |
| `RuudSend(aIndex, ammount)` | 2 | aIndex:int, ammount:int | nil | adds Ruud to the player and resends it (clamped 0..2e9) |
| `MonsterCreate(MonsterId, world, x, y, dir, Elementalatribute?)` | 5+ | MonsterId:int, world:int, x:int, y:int, dir:int, Elementalatribute?:int | int (monster entry) | spawns a monster at a location (optional elemental attribute) |
| `MonsterCreateEx(MonsterId, world, x, y, dir, MoveRange, Life, Damage, Defense, AttackRate, DefenseRate, ExperienceRate, Elementdamage, Elementdefense, Elementalatribute?)` | 14+ | MonsterId:int, world:int, x:int, y:int, dir:int, MoveRange:int, Life:int, Damage:int, Defense:int, AttackRate:int, DefenseRate:int, ExperienceRate:int, Elementdamage:int, Elementdefense:int, Elementalatribute?:int | int (monster entry) | spawns a monster with custom stat overrides |
| `MonsterDelete(aIndex)` | 1 | aIndex:int | nil | removes a monster/object from the world |
| `MonsterSummonCreate(aIndex, MonsterId, bValue, cValue, dValue, eValue, fValue, Elementalatribute?)` | 7+ | aIndex:int, MonsterId:int, bValue:int, cValue:int, dValue:int, eValue:int, fValue:int, Elementalatribute?:int | int (monster entry) | creates a summoned pet monster owned by the player |
| `MonsterSummonDelete(aIndex)` | 1 | aIndex:int | nil | removes the player's summoned monster |
| `MoveUser(aIndex, Gate)` | 2 | aIndex:int, Gate:int | nil | moves the player to a gate |
| `MoveUserEx(aIndex, WorldId, x, y)` | 4 | aIndex:int, WorldId:int, x:int, y:int | nil | teleports the player to a map coordinate |
| `MessageSend(aIndex, type, title, aString)` | 4 | aIndex:int, type:int, title:string, aString:string | nil | sends a message box to one player |
| `MessageSendToAll(type, title, aString)` | 3 | type:int, title:string, aString:string | nil | sends a message box to all online players |
| `NoticeSend(aIndex, type, aString)` | 3 | aIndex:int, type:int, aString:string | nil | sends a notice to one player |
| `NoticeSendToAll(type, aString)` | 2 | type:int, aString:string | nil | sends a notice to all online players |
| `NoticeGlobalSend(type, aString)` | 2 | type:int, aString:string | nil | sends a cross-server (global) notice via server link |
| `PartyCreate(aIndex)` | 1 | aIndex:int | nil (1 on success) | creates a party led by the player |
| `PartyDelete(partyId)` | 1 | partyId:int | nil | deletes a party |
| `PartyAddMember(partyId, aIndex)` | 2 | partyId:int, aIndex:int | nil (1 on success) | adds a player to a party |
| `PartyDelMember(partyId, aIndex)` | 2 | partyId:int, aIndex:int | nil (1 on success) | removes a player from a party |
| `PartyGetMemberCount(aValue)` | 1 | aValue:int (partyId) | int (member count) | returns the number of members in a party |
| `PartyGetMemberIndex(PartyID, MemberID)` | 2 | PartyID:int, MemberID:int | int (slot index, -1 if not found) | returns a member's slot index within a party |
| `ObjectGetCoin(aIndex)` | 1 | aIndex:int | table {Coin1=credits, Coin2=goblinPoints} | returns the player's coin balances |
| `ObjectAddCoin(aIndex, credit, goblin, cValue)` | 4 | aIndex:int, credit:int, goblin:int, cValue:int | nil | adds credits and goblin points to the player |
| `ObjectSubCoin(aIndex, credit, goblin, cValue)` | 4 | aIndex:int, credit:int, goblin:int, cValue:int | nil | subtracts credits and goblin points from the player |
| `PermissionCheck(aIndex, aValue)` | 2 | aIndex:int, aValue:int | int (permission flag) | returns a player permission state by id (1..13) |
| `PermissionInsert(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | grants a player permission by id (1..13) |
| `PermissionRemove(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | revokes a player permission by id (1..13) |
| `PKLevelSend(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | sets and resends the player's PK level |
| `PostSend(type, messageid, aString, bString)` | 4 | type:int, messageid:int, aString:string, bString:string | nil | sends a server post/announce message (4 formats) |
| `QuestStateCheck(aIndex, QuestIndex)` | 2 | aIndex:int, QuestIndex:int | int (1 if evolution quest 6 complete, else 0) | checks a player's quest completion state |
| `RandomGetNumber(aValue)` | 1 | aValue:int | int (0..aValue-1) | returns a random number below the given bound |
| `SkinChangeSend(aIndex, Skin)` | 2 | aIndex:int, Skin:number | nil | changes the player's skin and recreates viewport |
| `UserDisconnect(aIndex)` | 1 | aIndex:int | nil | closes the player's socket (disconnect) |
| `UserGameLogout(aIndex, aValue)` | 2 | aIndex:int, aValue:int | nil | logs the player out with the given close type (clamped 0..2) |
| `UserCalcAttribute(aIndex)` | 1 | aIndex:int | bool (1) | Recalculates player's character stats |
| `UserInfoSend(aIndex)` | 1 | aIndex:int | bool (1) | Sends the player's stats packet to client |
| `UserActionSend(aIndex, target, Action)` | 3 | aIndex:int, target:int, Action:int | bool (1) | Sends a player action/emote toward a target |
| `SQLAsyncQuery(query, label?, callbackParam?)` | 1+ | Query:string, Label:string?, CallBackParam:string? | bool (1) | Runs an async SQL query on a detached thread with callback (1-3 args) |
| `UserWarehouseOpen(aIndex, warehouseindex?)` | 1+ | aIndex:int, warehouseindex:int? | bool (1) | Opens warehouse interface (multi-warehouse/VIP aware), default index 1 |
| `MuunInventoryGetWearSize()` | 0 | (none) | int (MUUN_INVENTORY_WEAR_SIZE) | Returns muun inventory wear slot count |
| `MuunInventoryGetFullSize()` | 0 | (none) | int (MUUN_INVENTORY_SIZE) | Returns total muun inventory size |
| `MuunInventoryGetItemIndex(aIndex, aValue)` | 2 | aIndex:int, aValue:int (slot) | int (item index) | Returns the item index at a muun inventory slot |
| `MuunInventoryGetItemCount(aIndex, Item, level)` | 3 | aIndex:int, Item:int, level:number | int (count) | Counts matching items in muun inventory by index/level |
| `MuunInventoryGetItemTable(aIndex, aValue)` | 2 | aIndex:int, aValue:int (slot) | table (item fields) | Returns full item attribute table for a muun inventory slot |
| `MuunInventorySetItemTable(aIndex, Slot, itemTable)` | 3 | aIndex:int, Slot:int, table:table | bool (1) | Writes item fields from a table into a muun inventory slot and sends update |
| `MuunInventoryDelItemIndex(aIndex, slot)` | 2 | aIndex:int, slot:int | bool (1) | Deletes the item at a muun inventory slot and refreshes list |
| `MuunInventoryDelItemCount(aIndex, Item, level, count)` | 4 | aIndex:int, Item:int, level:number, count:int | bool (1) | Removes a count of matching items from muun inventory |
| `UserGetOptionTable(aIndex)` | 1 | aIndex:int | table (combat/stat options) | Returns a table of the player's computed combat/stat option values |
| `UserSetOptionTable(aIndex, optionTable)` | 2 | aIndex:int, table:table | bool (1) | Applies combat/stat option values from a table onto the player |
| `EffectAdd(aIndex, flag, buffid, bufftime, value1, value2, value3, value4)` | 8 | aIndex:int, flag:int, buffid:int, bufftime:int, value1:number, value2:number, value3:number, value4:number | nil | Adds a buff/effect to the target object, mapping buffid to the appropriate BuffEffect options and duration |
| `EffectDel(aIndex, buffid)` | 2 | aIndex:int, buffid:int | nil | Removes a buff from the player |
| `EffectCheck(aIndex, buffid)` | 2 | aIndex:int, buffid:int | int (HasBuff result) | Returns whether the player has the given buff |
| `EffectClear(aIndex)` | 1 | aIndex:int | nil | Clears all buffs on the player |
| `GetPlayerMac(aIndex)` | 1 | aIndex:int | string (MAC address) | Returns the player account's MAC address |
| `GetPlayerSerial(aIndex)` | 1 | aIndex:int | int (disk serial) | Returns the player account's disk serial |
| `GetMapName(aIndex)` | 1 | aIndex:int | string (map name) | Returns the name of the player's current world/map |
| `SetWindowText(aIndex, title)` | 2 | aIndex:int, title:string | nil | Sends custom window text to one player or, if not found, all playing characters |
| `GetObjectLastGainedExp(aIndex)` | 1 | aIndex:int | number (last gained exp) | Returns the player's last gained experience |
| `GetObjectMasterPoint(aIndex)` | 1 | aIndex:int | int (master points) | Returns the player's master-level points |
| `GetObjectMajesticPoint(aIndex)` | 1 | aIndex:int | int (majestic points) | Returns the player's majestic-level points |
| `IsItem(ItemID)` | 1 | ItemID:int | bool (true if valid template) | Checks whether an item template exists for the ID |
| `IsSocketItem(ItemID)` | 1 | ItemID:int | bool | Checks whether the item is a socket-kind item |
| `IsElementalItem(ItemID)` | 1 | ItemID:int | bool | Checks whether the item is pentagram or errtel (elemental) |
| `IsPentagramItem(ItemID)` | 1 | ItemID:int | bool | Checks whether the item is a pentagram item |
| `Is28Option()` | 0 | (none) | bool (always false) | Stub that always returns false |
| `GetBagItemLevel(ItemMinLevel, ItemMaxLevel)` | 2 | ItemMinLevel:int, ItemMaxLevel:int | int (random level) | Returns a random item level in the given range |
| `GetAncientOpt(ItemID)` | 1 | ItemID:int | int (ancient option value) | Returns a random ancient option for the item |
| `GetObjectTotalStrength(aIndex)` | 1 | aIndex:int | int (total strength) | Returns the player's total (effective) strength |
| `GetObjectTotalDexterity(aIndex)` | 1 | aIndex:int | int (total agility) | Returns the player's total agility/dexterity |
| `GetObjectTotalVitality(aIndex)` | 1 | aIndex:int | int (total vitality) | Returns the player's total vitality |
| `GetObjectTotalEnergy(aIndex)` | 1 | aIndex:int | int (total energy) | Returns the player's total energy |
| `GetObjectTotalLeadership(aIndex)` | 1 | aIndex:int | int (total leadership) | Returns the player's total leadership |
| `GetRandomValue(aIndex)` | 1 | aIndex:int (upper bound) | int (random 0..aIndex) | Returns a random integer between 0 and the argument |
| `AddItemBag(BagType, BagID, ItemID, ItemLevel)` | 4 | BagType:int, BagID:int, ItemID:int, ItemLevel:int | nil | Adds an item entry to an item bag |
| `CreateItem(UseType, MonsterIndex, MapNumber, MonsterX, MonsterY, ItemID, ItemLevel, ItemDur, IsSkill, IsLuck, IsOption, PlayerIndex, IsAncient, Duration, IsSocket, IsElemental, MuunEvoItemID, Exc, MasteryExc, Socket1, Socket2, Socket3, Socket4, Socket5, SocketBonus, ErrtelRank, Count)` | 27 | UseType:int, MonsterIndex:int, MapNumber:int, MonsterX:int, MonsterY:int, ItemID:int, ItemLevel:int, ItemDur:int, IsSkill:int, IsLuck:int, IsOption:int, PlayerIndex:int, IsAncient:int, Duration:int, IsSocket:int, IsElemental:int, MuunEvoItemID:int, Exc:int, MasteryExc:int, Socket1:int, Socket2:int, Socket3:int, Socket4:int, Socket5:int, SocketBonus:int, ErrtelRank:int, Count:int | int (ItemID on success, -1/0 on failure) | Builds a fully-specced item and drops it on the map or inserts into inventory/chaos box/Gremory case |
| `InsertItem_GremoryCase(PlayerIndex, GremoryCaseType, GremoryCaseGiveType, ItemID, ItemLevel, ItemDur, IsSkill, IsLuck, IsOption, IsAncient, IsSocket, IsElemental, MuunEvoItemID, Exc, MasteryExc, Socket1, Socket2, Socket3, Socket4, Socket5, SocketBonus, ReceiptDuration, Duration, Count)` | 24 | PlayerIndex:int, GremoryCaseType:int, GremoryCaseGiveType:int, ItemID:int, ItemLevel:int, ItemDur:int, IsSkill:int, IsLuck:int, IsOption:int, IsAncient:int, IsSocket:int, IsElemental:int, MuunEvoItemID:int, Exc:int, MasteryExc:int, Socket1:int, Socket2:int, Socket3:int, Socket4:int, Socket5:int, SocketBonus:int, ReceiptDuration:int, Duration:int, Count:int | int (ItemID, -1 if no player) | Builds an item and inserts it into the player's Gremory Case |
| `InsertItem_GremoryCaseEx(PlayerIndex, GremoryCaseType, GremoryCaseGiveType, ItemID, ItemLevel, ItemDur, IsSkill, IsLuck, IsOption, IsAncient, Item380, MuunEvoItemID, Exc, MasteryExc, Socket1, Socket2, Socket3, Socket4, Socket5, SocketBonus, ReceiptDuration, Count)` | 22 | PlayerIndex:int, GremoryCaseType:int, GremoryCaseGiveType:int, ItemID:int, ItemLevel:int, ItemDur:int, IsSkill:int, IsLuck:int, IsOption:int, IsAncient:int, Item380:int, MuunEvoItemID:int, Exc:int, MasteryExc:int, Socket1:int, Socket2:int, Socket3:int, Socket4:int, Socket5:int, SocketBonus:int, ReceiptDuration:int, Count:int | int (ItemID, -1 if no player) | Builds an item (with 380 option) and inserts it into the player's Gremory Case |
| `GCTotalFreeSlotCount(aIndex, CaseType)` | 2 | aIndex:int, CaseType:int | int (free slots, -1 on error) | Returns total free slots in the player's Gremory Case of the given type |
| `GetObjectMaxNormalPoints(aIndex)` | 1 | aIndex:int | int (max normal points, -1 if no player) | Returns the player's max normal stat points |
| `ResetMajesticTree(aIndex)` | 1 | aIndex:int | nil (-1 if no player) | Resets the player's majestic skill tree |
| `SendRgbChat(aIndex, text)` | 2 | aIndex:int, text:string | nil (-1 if no object) | Stub intended to send an RGB-colored chat/notice (currently no-op) |
| `GetObjectPlayerID(aIndex)` | 1 | aIndex:int | int (GUID, -1 if no player) | Returns the player's GUID/player ID |
| `CalculateCharacter(aIndex)` | 1 | aIndex:int | nil (-1 if no player) | Recalculates the player's character stats |
| `UpdatePlayerLevels(aIndex)` | 1 | aIndex:int | nil (-1 if no player) | Recomputes/updates the player's levels |
| `ExpSend(aIndex, Exp)` | 2 | aIndex:int, Exp:number | nil (-1 if no player) | Grants the player experience |
