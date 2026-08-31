# MU Plugin Framework (MUPF)

## Server-managed Lua plugins with an in-game client UI

The **MU Plugin Framework (MUPF)** lets a server operator add new client-facing features without rebuilding the game client for every plugin and without asking players to install separate plugin DLLs.

The game client contains only the **generic MUPF host** built into Main.dll. It does **not** contain any individual plugin package before the player connects. After login, the GameServer selects the authorized plugins and streams only their approved client UI into memory. The plugin's Lua logic, permissions, data access, and database operations remain on the GameServer.

Each plugin is a self-contained module made of three parts:

- **Manifest (`plugin.json`)** — identifies the plugin, defines its UI, API version, entry points, limits, and requested permissions.
- **Server logic (`server/server.lua`)** — runs only inside the GameServer and can use approved player or database operations. It is never delivered to players.
- **Client UI (`client/`)** — HTML, CSS, JavaScript, SVG, and other approved assets that are streamed by the GameServer and rendered in-game by the client plugin host.

![MUPF client and server architecture](assets/mupf-plugin-system-block-diagram.png)

## How it works

1. **The operator installs the plugin.** During development it can be a source folder. For production it is distributed as a single encrypted `.mupf` package under `Data/ClientLuaPlugin/`.
2. **The GameServer validates the plugin.** It reads the manifest, checks the supported host API version, applies package and message limits, and calculates the final capability set.
3. **Permissions are granted by policy.** A capability is available only when it is requested by the plugin manifest and also allowed by the server operator in `policy.json`. Everything else is denied by default.
4. **The server-side Lua is loaded privately.** `server.lua` is loaded into the GameServer Lua runtime. It remains on the server and is never included in the client package.
5. **Eligible client files are delivered automatically.** When a player connects, the GameServer sends a `BEGIN` handshake, the plugin package in validated chunks, and a final `READY` message.
6. **The client creates an isolated plugin instance.** Main.dll reassembles the package, validates its size and API requirements, and unpacks the client assets into a per-plugin virtual file-system sandbox. No native plugin DLL is loaded on the player's computer.
7. **The UI becomes available in-game.** A plugin can open through a launcher button, a configured hotkey, or a server request. Its HTML/CSS/JavaScript UI is rendered in a separate window that follows the game window.
8. **Client and server communicate through a controlled bridge.** The UI calls `MUPF.invoke(function, data)`. The GameServer routes the request only to that plugin's Lua `OnInvoke` handler. The handler validates the request, performs allowed work, and sends a correlated reply or an optional server push.
9. **The UI updates from live server data.** The client receives the result, calls the registered JavaScript callback, and re-renders the plugin window. This supports rankings, account tools, event panels, dashboards, and other live features.

## Request and response flow

```text
Player action
    -> MUPF.invoke(function, JSON)
    -> secure plugin protocol
    -> GameServer Plugin Manager
    -> server.lua: OnInvoke(ctx, function, args, requestId)
    -> approved player / database operation
    -> ctx:reply(...) or ctx:push(...)
    -> client callback
    -> UI re-render
```

## Security and isolation

- **Deny-by-default permissions:** effective capabilities are the intersection of the manifest request and the operator's `policy.json`.
- **Server code stays private:** `server.lua`, database access, and server-only implementation details never leave the GameServer.
- **No third-party native code:** plugins do not inject their own DLLs into the game client.
- **Per-plugin isolation:** every plugin receives its own numeric routing ID, state, request queue, UI window, and virtual file system.
- **Compatibility gate:** plugins that require a newer host API are quarantined instead of being executed incorrectly.
- **Resource limits:** package size, message size, pending requests, request timeouts, and server-side rate limits protect both the client and GameServer.
- **Controlled database access:** SQL is asynchronous, parameterized, and available only when the operator explicitly grants the required capability.
- **Failure containment:** malformed packages, invalid messages, unsupported versions, and client script errors are rejected or contained without enabling unrestricted access to the game process.

## Player experience

From the player's perspective, plugins are automatic:

- no separate plugin installer;
- no manual UI-file copying;
- no extra third-party DLLs;
- plugins appear only after the server authorizes and delivers them;
- launcher buttons and windows follow the game window and hide outside the active game world;
- the GameServer remains the authority for all important data and actions.

## Development and production

### Development mode

- Keep the plugin as a source folder under `Data/ClientLuaPlugin/`.
- Enable `MUPF_DEV_HOTRELOAD` while developing.
- Edit client HTML/CSS/JavaScript and relog to receive the refreshed client package.
- Reload scripts or restart the GameServer after changing `server.lua`.

### Production mode

- Pack the complete plugin with **MupfPacker** into one encrypted `.mupf` file.
- Set `MUPF_DEV_HOTRELOAD = 0` so packages are loaded and cached normally.
- Grant only the minimum required capabilities in `policy.json`.
- Use the GameServer's **Reload Script** command to rescan server-side plugin content; connected players receive refreshed client assets on their next login/relog.

## In one sentence

**MUPF keeps plugin authority and Lua logic on the GameServer, automatically delivers only the approved UI assets to the client, and connects both sides through a permission-controlled request/reply protocol.**
