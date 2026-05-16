# Scheduler

`OnTimerThread` fires every **1 second**. Use it for all recurring work.

---

## Tick Counter

```lua
local tick = 0
BridgeFunctionAttach("OnTimerThread", function()
    tick = tick + 1
    if tick % 5    == 0 then  end  -- every 5 sec
    if tick % 60   == 0 then  end  -- every minute
    if tick % 3600 == 0 then  end  -- every hour
end)
```

---

## Wall-Clock (specific time)

```lua
BridgeFunctionAttach("OnTimerThread", function()
    local t = os.date("*t")

    -- Every minute at :00
    if t.sec == 0 then end

    -- Daily at midnight
    if t.hour == 0 and t.min == 0 and t.sec == 0 then
        DailyReset()
    end
end)
```

---

## Daily Reset Pattern

```lua
local lastResetDate = ""

BridgeFunctionAttach("OnTimerThread", function()
    local t     = os.date("*t")
    local today = os.date("%Y-%m-%d")
    if t.hour == 0 and t.min == 0 and t.sec <= 1 and lastResetDate ~= today then
        lastResetDate = today
        NoticeSendToAll(0, "Daily rewards reset!")
        -- reset tables, flush DB, etc.
    end
end)
```

---

## Per-Player Cooldown

```lua
local cooldowns = {}

BridgeFunctionAttach("OnCommandManager", function(aIndex, command)
    if command ~= "/reward" then return 0 end
    local now = os.time()
    if (now - (cooldowns[aIndex] or 0)) < 3600 then
        local rem = 3600 - (now - (cooldowns[aIndex] or 0))
        NoticeSend(aIndex, 0, "Cooldown: " .. rem .. "s remaining.")
        return 1
    end
    cooldowns[aIndex] = now
    -- give reward...
    return 1
end)

BridgeFunctionAttach("OnCharacterClose", function(aIndex)
    cooldowns[aIndex] = nil
end)
```

---

## Periodic Broadcast

```lua
local msgs = { "Join our Discord!", "Events at 20:00!", "Top resets win prizes!" }
local msgTick = 0
local msgIdx  = 1

BridgeFunctionAttach("OnTimerThread", function()
    msgTick = msgTick + 1
    if msgTick % 300 ~= 0 then return end  -- every 5 minutes
    NoticeSendToAll(0, msgs[msgIdx])
    msgIdx = (msgIdx % #msgs) + 1
end)
```

---

## Periodic DB Flush

```lua
local dirty = {}  -- [charName] = data

local function markDirty(aIndex, data)
    dirty[GetObjectName(aIndex)] = data
end

local flushTick = 0
BridgeFunctionAttach("OnTimerThread", function()
    flushTick = flushTick + 1
    if flushTick % 30 ~= 0 then return end
    for name, d in pairs(dirty) do
        SQLAsyncQuery("flush", string.format(
            "UPDATE myplugin SET points=%d WHERE char_name='%s'",
            d.points, name:gsub("'", "''")))
    end
    dirty = {}
end)

BridgeFunctionAttach("OnCharacterClose", function(aIndex)
    local name = GetObjectName(aIndex)
    if dirty[name] then
        SQLAsyncQuery("flush", string.format(
            "UPDATE myplugin SET points=%d WHERE char_name='%s'",
            dirty[name].points, name:gsub("'", "''")))
        dirty[name] = nil
    end
end)
```

---

## Fixed Schedule (multiple times per day)

```lua
local schedule = { {12,0}, {20,0}, {22,30} }
local fired = {}

BridgeFunctionAttach("OnTimerThread", function()
    local t = os.date("*t")
    if t.sec ~= 0 then return end
    local key = t.hour .. ":" .. t.min
    for _, s in ipairs(schedule) do
        if t.hour == s[1] and t.min == s[2] and not fired[key] then
            fired[key] = true
            NoticeSendToAll(1, "Event starting!")
        end
    end
    if t.hour == 0 and t.min == 0 then fired = {} end  -- midnight reset
end)
```

---

## Quick Reference

| Goal | Method |
|---|---|
| Every N seconds | `tick % N == 0` |
| Specific time | `os.date("*t")` + guard flag |
| Player cooldown | `os.time()` stored in table |
| Daily reset | compare `os.date("%Y-%m-%d")` |
| Periodic DB write | accumulate dirty, flush on tick |
