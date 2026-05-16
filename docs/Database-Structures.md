# Database Structures

All SQL is **asynchronous** — queries run on a background thread, results arrive via callback. Never blocks the game loop.

---

## Basic Pattern

```lua
-- 1. Fire the query
SQLAsyncQuery("my_label", "SELECT col FROM table WHERE id = 1")

-- 2. Handle result
BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if label ~= "my_label" then return end

    if not rows[1] then return end       -- no results

    local value = rows[1]["col"]         -- always a string
    local num   = tonumber(rows[1]["col"] or 0)
end)
```

---

## `SQLAsyncQuery(label, sql)`

| Param | Description |
|---|---|
| `label` | Identifier returned in `OnSQLAsyncResult` — use unique prefix per plugin |
| `sql` | Full SQL statement |

---

## `OnSQLAsyncResult(label, rows)`

| Param | Description |
|---|---|
| `label` | Label from the query |
| `rows` | Array of row tables. `rows[1]` = first row, `rows[n]` = nth row |

- All column values are **strings** — use `tonumber()` for numbers
- `rows[1]` is `nil` when no rows returned

---

## SQL Injection — Always Escape

```lua
local name = GetObjectName(aIndex):gsub("'", "''")
local sql = string.format("SELECT * FROM t WHERE char_name = '%s'", name)
```

---

## Passing Player Context to Callback

The callback does not receive `aIndex`. Embed it in the label:

```lua
-- On login: fire query with index in label
BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    local name = GetObjectName(aIndex):gsub("'", "''")
    SQLAsyncQuery("load_" .. aIndex,
        string.format("SELECT points FROM data WHERE char_name = '%s'", name))
end)

-- In callback: extract index from label
BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if not label:find("load_", 1, true) then return end
    local aIndex = tonumber(label:sub(6))

    -- Player may have logged out by now — always check
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end

    local points = tonumber(rows[1] and rows[1]["points"] or 0)
end)
```

---

## Common Query Patterns

### SELECT one row
```lua
SQLAsyncQuery("load_" .. aIndex, string.format(
    "SELECT points, last_login FROM player_data WHERE char_name = '%s'",
    GetObjectName(aIndex):gsub("'", "''")))
```

### INSERT (no result needed)
```lua
SQLAsyncQuery("insert_nores", string.format(
    "INSERT INTO kill_log (char_name, monster_id, killed_at) VALUES ('%s', %d, NOW())",
    name, monsterId))
```

### UPSERT (insert or update)
```lua
SQLAsyncQuery("save_nores", string.format(
    "INSERT INTO player_data (char_name, points) VALUES ('%s', %d) "
 .. "ON DUPLICATE KEY UPDATE points = %d",
    name, points, points))
```

### Multiple rows
```lua
SQLAsyncQuery("top10", "SELECT char_name, points FROM board ORDER BY points DESC LIMIT 10")

BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if label ~= "top10" then return end
    for i, row in ipairs(rows) do
        LogPrint(string.format("#%d %s — %s pts", i, row["char_name"], row["points"]))
    end
end)
```

---

## Recommended Table Design

```sql
CREATE TABLE myplugin_data (
    char_name  VARCHAR(10)  NOT NULL,
    points     INT          NOT NULL DEFAULT 0,
    last_claim DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (char_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

---

## Key Rules

| Rule | Why |
|---|---|
| Always use `SQLAsyncQuery` | No blocking SQL from Lua |
| Re-check `GetObjectConnected()` in callback | Player may disconnect before result arrives |
| Use unique labels per plugin | Prevent label collisions |
| `tonumber()` all numeric columns | All values arrive as strings |
| Escape `'` in strings | Prevent SQL injection |
