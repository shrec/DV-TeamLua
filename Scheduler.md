# Scheduler

The server fires `OnTimerThread` once per second. Use this as your primary scheduling mechanism — everything from cooldown tracking to daily resets runs through it.

---

## Basic Timer

```lua
local tick = 0

BridgeFunctionAttach("OnTimerThread", function()
    tick = tick + 1

    if tick % 5  == 0 then -- every 5 seconds
    end

    if tick % 60 == 0 then -- every minute
    end

    if tick % 3600 == 0 then -- every hour
    end
end)
```

`tick` resets to 0 when scripts reload. If exact wall-clock timing matters, use `os.time()` instead.

---

## Wall-Clock Scheduling

```lua
BridgeFunctionAttach("OnTimerThread", function()
    local t = os.date("*t")

    -- once per minute at second :00
    if t.sec == 0 then
        -- runs at 12:00:00, 12:01:00, 12:02:00 …
    end

    -- daily at midnight (guard with a flag to prevent double-fire)
    if t.hour == 0 and t.min == 0 and t.sec == 0 then
        DailyReset()
    end
end)
```

---

## Daily Reset Pattern

The most common pattern in real plugins — resets a flag once per day at 00:00:

```lua
local DailyPlugin = {}
local lastResetDate = ""

local function DoDailyReset()
    -- broadcast notice
    NoticeSendToAll(0, "Daily rewards have been reset!")
    -- reset in-memory tables, push SQL writes, etc.
end

BridgeFunctionAttach("OnTimerThread", function()
    local t      = os.date("*t")
    local today  = os.date("%Y-%m-%d")

    if t.hour == 0 and t.min == 0 and t.sec <= 1 and lastResetDate ~= today then
        lastResetDate = today
        DoDailyReset()
    end
end)
```

The `sec <= 1` guard allows a 1-second window in case the timer fires slightly late.

---

## Per-Player Cooldown

```lua
local cooldowns = {}   -- [aIndex] = os.time() of last use

BridgeFunctionAttach("OnCommandManager", function(aIndex, command)
    if command ~= "/reward" then return 0 end

    local now = os.time()
    local cd  = cooldowns[aIndex] or 0

    if now - cd < 3600 then  -- 1-hour cooldown
        local remaining = 3600 - (now - cd)
        NoticeSend(aIndex, 0, string.format("Cooldown: %d seconds remaining.", remaining))
        return 1
    end

    cooldowns[aIndex] = now
    -- give reward
    return 1
end)

-- Clean up on logout to avoid memory leak
BridgeFunctionAttach("OnCharacterClose", function(aIndex)
    cooldowns[aIndex] = nil
end)
```

---

## Scheduled Broadcast

```lua
local announcements = {
    "Visit our Discord!",
    "Events run every day at 20:00.",
    "Top resets get bonus prizes!",
}
local announceTick = 0
local announceIdx  = 1

BridgeFunctionAttach("OnTimerThread", function()
    announceTick = announceTick + 1
    if announceTick % 300 ~= 0 then return end  -- every 5 minutes

    NoticeSendToAll(0, announcements[announceIdx])
    announceIdx = (announceIdx % #announcements) + 1
end)
```

---

## Scheduled SQL Flush

Write accumulated in-memory data to the database periodically instead of on every event:

```lua
local dirty = {}   -- [charName] = pendingData

-- Mark dirty when something changes
local function markDirty(aIndex, data)
    dirty[GetObjectName(aIndex)] = data
end

-- Flush every 30 seconds
local flushTick = 0
BridgeFunctionAttach("OnTimerThread", function()
    flushTick = flushTick + 1
    if flushTick % 30 ~= 0 then return end

    for charName, data in pairs(dirty) do
        SQLAsyncQuery("flush_noresult", string.format(
            "UPDATE myplugin SET points = %d WHERE char_name = '%s'",
            data.points, charName:gsub("'", "''")))
    end
    dirty = {}
end)

-- Also flush on logout
BridgeFunctionAttach("OnCharacterClose", function(aIndex)
    local name = GetObjectName(aIndex)
    if dirty[name] then
        SQLAsyncQuery("flush_noresult", string.format(
            "UPDATE myplugin SET points = %d WHERE char_name = '%s'",
            dirty[name].points, name:gsub("'", "''")))
        dirty[name] = nil
    end
end)
```

---

## Timed Events (Fixed Schedule)

```lua
local eventSchedule = {
    { hour = 12, min = 0  },   -- noon
    { hour = 20, min = 0  },   -- 20:00
    { hour = 22, min = 30 },   -- 22:30
}
local eventFired = {}

BridgeFunctionAttach("OnTimerThread", function()
    local t   = os.date("*t")
    local key = string.format("%d-%d-%d", t.hour, t.min, t.sec)

    if t.sec ~= 0 then return end  -- only check at :00 seconds

    for _, slot in ipairs(eventSchedule) do
        if t.hour == slot.hour and t.min == slot.min and not eventFired[key] then
            eventFired[key] = true
            NoticeSendToAll(1, "Event is starting now!")
            -- launch event
        end
    end

    -- clean old keys (prevent table growing forever)
    if t.min == 0 and t.hour == 0 then
        eventFired = {}
    end
end)
```

---

## Tips

| Goal | Approach |
|---|---|
| Run every N seconds | `tick % N == 0` counter |
| Run at specific time | `os.date("*t")` + guard flag |
| Player cooldowns | `os.time()` stored per `aIndex` |
| Periodic DB flush | Accumulate dirty table, flush on tick |
| Daily reset | `os.date("%Y-%m-%d")` compared to stored date |
| High-frequency (< 1s) | Not possible from Lua — handle in C++ |

Keep `OnTimerThread` fast. Avoid heavy loops over all players every tick — use counters to spread work.
