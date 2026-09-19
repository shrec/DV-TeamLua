# Database Structures

All SQL runs **asynchronously** on a background thread. The game loop is never blocked. Results arrive via `OnSQLAsyncResult`.

---

## Callback Signature

```lua
BridgeFunctionAttach("OnSQLAsyncResult", function(label, callbackParam, rows)
    -- label        = string you passed to SQLAsyncQuery
    -- callbackParam = optional extra string passed as 3rd arg to SQLAsyncQuery
    -- rows         = table (SELECT) or number (INSERT/UPDATE/DELETE or no results)
end)
```

> ⚠️ The callback receives **3 parameters** — `label`, `callbackParam`, `rows`. Many examples online only show 2. Missing `callbackParam` shifts `rows` into the wrong variable.

---

## `SQLAsyncQuery(sql [, label [, callbackParam]])`

The **SQL statement is the first argument**. The optional label is second;
`callbackParam` is third. To pass `callbackParam`, supply a label too.

| Parameter | Type | Description |
|---|---|---|
| `sql` | string | Full SQL statement to execute |
| `label` | string | Optional; returned as the first callback argument (empty string if omitted). Use a unique prefix per plugin. |
| `callbackParam` | string | Optional; returned unchanged as the second callback argument, useful for carrying player context |

```lua
SQLAsyncQuery("SELECT points FROM myplug WHERE char_name = 'Test'", "myplug_load")

-- with callbackParam to carry context
SQLAsyncQuery(sql, "myplug_load", tostring(aIndex))
```

---

## What `rows` Contains

### SELECT with results → table

```lua
-- rows is a 1-based Lua table:
-- rows[1] = first row  { colName = value, ... }
-- rows[2] = second row { colName = value, ... }
-- rows[n] = nth row

#rows   -- total number of rows returned
```

### SELECT with no results → number `0`

```lua
-- rows == 0  (not a table!)
```

The legacy query path also returns `0` after a failed SELECT. The callback
has no separate success/error argument, so `rows == 0` cannot prove that a
query succeeded with no rows. Do not base an irreversible charge, item removal,
or refund on that assumption.

### INSERT / UPDATE / DELETE → number (affected rows)

```lua
-- rows == number of rows affected
```

A failed write and a successful write affecting zero rows can likewise both
appear as `0` in this legacy callback.

**Always check the type before using:**

```lua
BridgeFunctionAttach("OnSQLAsyncResult", function(label, callbackParam, rows)
    if label ~= "myplug_load" then return end

    if type(rows) ~= "table" then
        -- no results (rows == 0) or write query
        return
    end

    -- safe to iterate
end)
```

---

## Column Types in Lua

The engine maps MySQL column types automatically:

| MySQL Type | Lua Type | Notes |
|---|---|---|
| `FLOAT`, `DOUBLE`, `DECIMAL` | number (float) | Use as-is |
| `TINYINT`, `SMALLINT`, `MEDIUMINT` | integer | Use as-is |
| `INT`, `BIGINT` | number (float) | Large int64 may lose precision |
| `VARCHAR`, `TEXT`, `CHAR` | string | Use `tonumber()` if numeric |
| `DATE`, `DATETIME`, `TIMESTAMP` | string | Format: `"2024-01-15 20:00:00"` |
| `NULL` | *(absent)* | Field is **not added** to the row table — access returns `nil` |

```lua
-- INT column → already a number, no conversion needed
local points = row["points"]            -- integer

-- VARCHAR column → string, convert if numeric
local level = tonumber(row["level"])    -- convert string → number

-- DATETIME column → string
local dt = row["created_at"]           -- "2024-01-15 20:00:00"

-- NULL column → nil (key not present)
if row["optional_col"] == nil then
    -- column was NULL in the database
end
```

---

## Reading Rows — All Patterns

### One row, one column

```lua
SQLAsyncQuery(string.format("SELECT points FROM myplug WHERE char_name = '%s'",
    GetObjectName(aIndex):gsub("'", "''")), "load_pts_" .. aIndex)

BridgeFunctionAttach("OnSQLAsyncResult", function(label, callbackParam, rows)
    if label:sub(1, 9) ~= "load_pts_" then return end
    local aIndex = tonumber(label:sub(10))

    if type(rows) ~= "table" then
        -- player not in table yet, first time
        return
    end

    local points = rows[1]["points"]   -- already a number (INT column)
end)
```

---

### One row, multiple columns

```lua
SQLAsyncQuery(string.format([[
    SELECT points, last_claim, vip_level, note
    FROM myplug
    WHERE char_name = '%s'
]], GetObjectName(aIndex):gsub("'", "''")), "load_player_" .. aIndex)

BridgeFunctionAttach("OnSQLAsyncResult", function(label, callbackParam, rows)
    if label:sub(1, 12) ~= "load_player_" then return end
    local aIndex = tonumber(label:sub(13))

    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
    if type(rows) ~= "table" then return end

    local row        = rows[1]
    local points     = row["points"]               -- INT  → number
    local lastClaim  = row["last_claim"]           -- DATETIME → string
    local vipLevel   = row["vip_level"]            -- TINYINT → integer
    local note       = row["note"]                 -- VARCHAR → string (or nil if NULL)

    LogPrint(string.format("[%s] pts=%d vip=%d last=%s",
        GetObjectName(aIndex), points, vipLevel, tostring(lastClaim)))
end)
```

---

### Multiple rows — leaderboard / top list

```lua
SQLAsyncQuery([[
    SELECT char_name, points, reset_count
    FROM myplug
    ORDER BY points DESC
    LIMIT 10
]], "top10")

BridgeFunctionAttach("OnSQLAsyncResult", function(label, callbackParam, rows)
    if label ~= "top10" then return end
    if type(rows) ~= "table" then
        LogPrint("Leaderboard is empty.")
        return
    end

    LogPrint("=== Top " .. #rows .. " Players ===")

    for i = 1, #rows do
        local row = rows[i]
        LogPrint(string.format("#%d %s — %d pts / %d resets",
            i, row["char_name"], row["points"], row["reset_count"]))
    end
end)
```

---

### Multiple rows — build a lookup table

Load all records into a Lua table on startup, then use it instantly without further DB calls:

```lua
local rewardTable = {}   -- [char_name] = { points, claimed }

BridgeFunctionAttach("OnReadScript", function()
    SQLAsyncQuery("SELECT char_name, points, claimed FROM myplug", "init_load_all")
end)

BridgeFunctionAttach("OnSQLAsyncResult", function(label, callbackParam, rows)
    if label ~= "init_load_all" then return end
    if type(rows) ~= "table" then return end

    for i = 1, #rows do
        local row = rows[i]
        rewardTable[row["char_name"]] = {
            points  = row["points"],
            claimed = row["claimed"],
        }
    end

    LogPrint("[MyPlugin] loaded " .. #rows .. " player records")
end)

-- Later: instant access, no DB call
BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    local name = GetObjectName(aIndex)
    local data = rewardTable[name]
    if data then
        NoticeSend(aIndex, 0, "Your points: " .. data.points)
    end
end)
```

---

### Multiple rows — process per-player rewards

```lua
SQLAsyncQuery([[
    SELECT char_name, item_cat, item_idx, item_level
    FROM pending_rewards
    WHERE delivered = 0
]], "pending_rewards")

BridgeFunctionAttach("OnSQLAsyncResult", function(label, callbackParam, rows)
    if label ~= "pending_rewards" then return end
    if type(rows) ~= "table" then return end

    for i = 1, #rows do
        local row      = rows[i]
        local name     = row["char_name"]
        local aIndex   = GetObjectIndexByName(name)

        if aIndex >= 0 and GetObjectConnected(aIndex) == OBJECT_ONLINE then
            InsertItem_GremoryCase(aIndex,
                row["item_cat"], row["item_idx"], row["item_level"],
                255, 0, 0, 0, 0, 0, 0, 0,
                0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)

            -- Mark delivered
            SQLAsyncQuery(string.format(
                "UPDATE pending_rewards SET delivered = 1 WHERE char_name = '%s'",
                name:gsub("'", "''")), "mark_delivered")
        end
    end
end)
```

---

## Passing Player Context

The callback does not know who triggered the query. Two patterns:

### Pattern A — Index in label (simple)

```lua
SQLAsyncQuery(sql, "load_" .. aIndex)

-- in callback:
local aIndex = tonumber(label:sub(6))   -- "load_" is 5 chars
if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
```

### Pattern B — callbackParam (cleaner for strings)

```lua
local name = GetObjectName(aIndex)
SQLAsyncQuery(sql, "load_data", name)   -- name passed as callbackParam

-- in callback:
local name   = callbackParam
local aIndex = GetObjectIndexByName(name)
if aIndex < 0 or GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
```

---

## Write Queries (INSERT / UPDATE / DELETE)

For non-critical, best-effort writes, you can send a query without using its
callback. The legacy callback cannot confirm whether a zero-row write failed:

```lua
-- INSERT
SQLAsyncQuery(string.format(
    "INSERT INTO myplug (char_name, points) VALUES ('%s', 0)",
    name:gsub("'", "''")), "w")

-- UPDATE
SQLAsyncQuery(string.format(
    "UPDATE myplug SET points = points + %d WHERE char_name = '%s'",
    amount, name:gsub("'", "''")), "my_update")

-- UPSERT (insert or update)
SQLAsyncQuery(string.format([[
    INSERT INTO myplug (char_name, points) VALUES ('%s', %d)
    ON DUPLICATE KEY UPDATE points = %d
]], name:gsub("'", "''"), points, points), "w")

-- DELETE
SQLAsyncQuery(string.format(
    "DELETE FROM myplug WHERE char_name = '%s'",
    name:gsub("'", "''")), "w")
```

For write queries, `rows` in the callback = number of affected rows. You can check it:

```lua
BridgeFunctionAttach("OnSQLAsyncResult", function(label, callbackParam, rows)
    if label ~= "my_update" then return end
    -- rows = affected row count (number)
    if rows == 0 then
        LogPrint("Update affected zero rows or failed; status is ambiguous")
    end
end)
```

---

## SQL Injection — Always Escape

```lua
-- Escape single quotes in any string that comes from a player or external source
local safe = GetObjectName(aIndex):gsub("'", "''")
local sql   = string.format("SELECT * FROM t WHERE char_name = '%s'", safe)
```

Never concatenate raw input directly into SQL.

---

## Quick Reference

| `rows` value | Meaning |
|---|---|
| `type(rows) == "table"` | SELECT returned rows |
| `rows == 0` | SELECT returned no rows |
| `type(rows) == "number"` and `rows > 0` | INSERT/UPDATE/DELETE affected N rows |

| Column MySQL type | Lua type |
|---|---|
| INT, TINYINT, SMALLINT, MEDIUMINT | integer (use directly) |
| BIGINT, FLOAT, DOUBLE, DECIMAL | number (use `math.floor()` for integers) |
| VARCHAR, TEXT, CHAR | string (use `tonumber()` if numeric) |
| DATE, DATETIME, TIMESTAMP | string (`"YYYY-MM-DD HH:MM:SS"`) |
| NULL | `nil` (field absent from row table) |
