# Database Structures

The server provides **asynchronous SQL** — queries execute on a background thread and results arrive via a callback. This means you never block the game loop.

---

## The Async SQL Pattern

```lua
-- 1. Fire the query (returns immediately)
SQLAsyncQuery("my_label", "SELECT col FROM table WHERE id = 1")

-- 2. Results arrive here later
BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if label ~= "my_label" then return end

    if not rows[1] then
        -- no rows returned
        return
    end

    local value = rows[1]["col"]
end)
```

The `label` is a free-form string you choose to identify which query returned. Use a unique prefix per plugin to avoid collisions.

---

## `SQLAsyncQuery(label, sql)`

| Parameter | Type | Description |
|---|---|---|
| `label` | string | Identifier passed back to `OnSQLAsyncResult` |
| `sql` | string | Full SQL statement |

```lua
SQLAsyncQuery("reward_check",
    "SELECT last_claim FROM daily_reward WHERE char_name = 'TestPlayer'")
```

---

## `OnSQLAsyncResult(label, rows)`

| Parameter | Type | Description |
|---|---|---|
| `label` | string | The label from `SQLAsyncQuery()` |
| `rows` | table | Array of row tables. Each row is `{columnName = value, ...}` |

- `rows[1]` = first row (check for `nil` before using)
- `rows[n]` = nth row
- Column values are always **strings** — use `tonumber()` when needed

```lua
BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if label ~= "reward_check" then return end

    local row = rows[1]
    if not row then
        -- no record found
        return
    end

    local lastClaim = row["last_claim"]          -- string
    local count     = tonumber(row["count"] or 0) -- convert to number
end)
```

---

## Passing Player Context

The callback does **not** receive `aIndex` — you must capture it in a closure or embed it in the label.

### Option A — Embed index in label (simple)
```lua
BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    SQLAsyncQuery("daily_load_" .. aIndex,
        string.format("SELECT * FROM daily WHERE char = '%s'",
            GetObjectName(aIndex):gsub("'", "''")))
end)

BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    local prefix = "daily_load_"
    if not label:find(prefix, 1, true) then return end

    local aIndex = tonumber(label:sub(#prefix + 1))
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end

    -- use rows here, player is still online
end)
```

### Option B — Cache in a Lua table (cleaner for complex state)
```lua
local pendingLoads = {}

BridgeFunctionAttach("OnCharacterEntry", function(aIndex)
    local name = GetObjectName(aIndex)
    pendingLoads[name] = { aIndex = aIndex, loadTime = os.time() }
    SQLAsyncQuery("daily_load",
        string.format("SELECT * FROM daily WHERE char = '%s'", name:gsub("'", "''")))
end)

BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if label ~= "daily_load" then return end
    local row = rows[1]
    if not row then return end

    local ctx = pendingLoads[row["char"]]
    if not ctx then return end
    pendingLoads[row["char"]] = nil

    local aIndex = ctx.aIndex
    if GetObjectConnected(aIndex) ~= OBJECT_ONLINE then return end
    -- proceed
end)
```

---

## SQL Injection Prevention

Always escape single quotes in strings that come from player input:

```lua
local name = GetObjectName(aIndex):gsub("'", "''")
local sql = string.format("SELECT * FROM tbl WHERE char_name = '%s'", name)
```

Never concatenate raw user input into SQL without escaping.

---

## Common Query Patterns

### SELECT one row
```lua
SQLAsyncQuery("load_" .. aIndex, string.format(
    "SELECT points, last_login FROM player_data WHERE char_name = '%s'",
    GetObjectName(aIndex):gsub("'", "''")))

-- in callback:
local points    = tonumber(rows[1] and rows[1]["points"] or 0)
local lastLogin = rows[1] and rows[1]["last_login"] or "never"
```

### INSERT or UPDATE (no result expected)
```lua
-- INSERT with ON DUPLICATE KEY UPDATE (upsert)
SQLAsyncQuery("save_noresult", string.format(
    "INSERT INTO player_data (char_name, points) VALUES ('%s', %d) "
    .. "ON DUPLICATE KEY UPDATE points = %d",
    name, points, points))

-- No callback needed for write-only queries — use a no-op label
-- or just ignore results in OnSQLAsyncResult
```

### INSERT only
```lua
SQLAsyncQuery("insert_noresult", string.format(
    "INSERT INTO kill_log (char_name, monster_id, killed_at) VALUES ('%s', %d, NOW())",
    name, monsterId))
```

### Multiple rows
```lua
BridgeFunctionAttach("OnSQLAsyncResult", function(label, rows)
    if label ~= "top10" then return end

    for i, row in ipairs(rows) do
        LogPrint(string.format("#%d: %s = %s pts", i, row["char_name"], row["points"]))
    end
end)

SQLAsyncQuery("top10", "SELECT char_name, points FROM leaderboard ORDER BY points DESC LIMIT 10")
```

---

## Table Design Tips

Keep plugin tables simple and always include `char_name` as the primary key so queries are fast without joins:

```sql
CREATE TABLE myplugin_data (
    char_name   VARCHAR(10)  NOT NULL,
    points      INT          NOT NULL DEFAULT 0,
    last_claim  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (char_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

---

## Important Caveats

- **No blocking SQL** — never attempt a blocking/synchronous DB call from Lua. Use `SQLAsyncQuery` exclusively.
- **Callback may be delayed** — the player could disconnect between the query and the result. Always re-check `GetObjectConnected()` inside the callback.
- **Label uniqueness** — if two queries share the same label, both results arrive with that label. Use specific labels or index-embedded labels to differentiate.
- **Column values are strings** — even numeric columns come back as strings. Always use `tonumber()`.
