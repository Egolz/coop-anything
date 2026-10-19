# Forager — agent notes

Working notes for adding co-op to Forager. Written incrementally per AGENTS.md rule 3.
Game slug: `forager`. Design proposal lives in `DESIGN.md`; one-page write-up in
`README.md`.

Owner-specific state (their install, saves, license, backups) lives in
`local/owner-state.md`, which git ignores. This file is PR-safe.

## Identity

| Field | Value |
|---|---|
| Game | Forager (2019), HopFrog / Humble Grove |
| Stores | Steam app **751780**, GOG, consoles |
| Engine | **GameMaker Studio 2** — confirmed by `data.win` in the install folder |
| Modding | built-in modding API; mods are `.gml` script folders published on Steam Workshop |
| OS tested | Windows (win32) |
| Money | **Modding is not Unity.** No BepInEx / Harmony / Il2Cpp. See "How modding works". |

## Observations (checked)

### The official multiplayer is dead
- Announced, beta-tested, then **cancelled in 2021** because it was too buggy; refunds
  offered to backers who bought for multiplayer.
  Sources: https://www.rockpapershotgun.com/forager-cancels-multiplayer ,
  https://techraptor.net/gaming/news/forager-developers-respond-to-multiplayer-cancellation

### The modding API and the cancelled netcode share an author
- **YellowAfterlife (yal.cc)** wrote Forager's modding API and helped with its networking.
  Sources: https://yal.cc/2019-in-review/ , https://yal.cc/2020-in-review/
- His retrospective on the cancelled multiplayer is the authoritative "why":
  https://yal.cc/forager-retrospect/ (mirror:
  https://web.archive.org/web/20250123110432/https://yal.cc/forager-retrospect/ ).
  Key quote: *"if the work started from local multiplayer, the game could still be played
  over internet using Parsec, Remote Play Together, or a tool I made. But it was not, so
  it cannot be, and at most someone might make a slightly more advanced version of
  topherlicious' PenPal mod."*

### Generic GameMaker co-op injectors do NOT apply
- **GMTogether** (also by YellowAfterlife, https://yellowafterlife.itch.io/gmtogether ,
  readme https://yal.cc/docs/gm/gmtogether/ ) injects online play into GameMaker games,
  but it is explicitly *"a tool for playing **local multiplayer** GameMaker games over the
  internet"*. Forager has **no local multiplayer**, so it is out — consistent with the
  retrospective quote above. (Forager is not in GMTogether's tested-games sheet as far as
  we could see; the sheet export returned only its README tab.)

### Prior art found by us, on Workshop
- **PencilPal (BETA)** — the co-op mod, Workshop item `2843162754`.
  - Its Workshop page states: *"This item has been removed from the community because it
    violates Steam Community & Content Guidelines"* and *"This item is incompatible with
    Forager"*. Subscription is no longer possible; page visible to existing subscribers.
  - Last updated **2023-04-08**; author "Two" (SteamID64 `76561198160202307`); target
    API version 1.0.1 (from `fmConfig.json`).
  - Comments into late 2025: frequent crashes, "unplayable", "full of bugs".
  - Needs the host's **TCP port 6511 forwarded**; no relay. Community workaround: Hamachi.
  - Predecessors: **PenPal** (topherlicious) which "only showed a friend's sprite".
  - Sources: https://steamcommunity.com/sharedfiles/filedetails/?id=2843162754 ,
    https://playco-opgame.com/multiplayer-mod/forager-multiplayer-mod/
- **Foragipelago** — a Forager mod that integrates with Archipelago. Proof that Forager
  mods are able to talk to **an external server over sockets**. (Found in the Workshop
  listing; item id 3760668082 — page did not render through our fetcher, verify later.)

### The hub has no prior work on Forager
- No Forager row in `data/directory.yaml`; no issues/PRs mentioning Forager in
  cranebeaker/coop-anything (checked 2026-10-19 via GitHub search UI).

## How modding works (from PencilPal source, observed)

A Forager mod is a **folder of `.gml` files** compiled by the game's own modding system.
PencilPal's folder (still in the owner's Steam Workshop cache) is a complete reference
implementation; the file list was:

```
fmConfig.json          manifest: {name, author, targetAPIVersion, dependencies}
Main.gml               #define Main — entry point, globals, setup
Networking.gml         #define Host / Join / Disconnect / packets (47 KB)
GUI.gml                in-game phone UI (18 KB)
Math.gml
Events/OnStep.gml      per-step hook (33 KB) — the sync loop
Events/OnEvent.gml     (15 KB)
Events/OnNetworkAsync.gml   async networking callbacks
Events/OnDrawEnd.gml, OnDrawGUIEnd.gml, OnChatMessage.gml, OnLevelUp.gml
Resources/Sprites/..., Resources/Sounds/...
```

### Modding API surface seen in the code
- Game data: `Spawn.Count`, `SpawnGet(l, SpawnData.ID)`, `Structure.Count`,
  `StructureGet(l, StructureData.Object)`, `StructureCreate(...)`, `ZoneCreate(...)`,
  enums `StructureType`, `ZoneType`, `BuildMenuCategory`, item enum `Item.*` (e.g.
  `Item.Steel`, `Item.Coin`), `spr*`/`tiles*` built-in asset names.
- Player/world: `objPlayer`, `objPlayer.room`, `objPlayer.sprite_index`, `image_index`,
  `ItemCount(Item.*)`, `RemoveItem(Item.*, n)`.
- Persistence: **`ModSaveDataSubmit` / `ModSaveDataFetch`** — used to sync treasure chests.
- Utils: `Trace(...)` (logging), `get_string` / `get_integer` (GM dialogs),
  `sprite_add`, `audio_create_stream`.
- Event hooks the API calls into: **`OnStep`, `OnEvent`, `OnDrawEnd`, `OnDrawGUIEnd`,
  `OnChatMessage`, `OnLevelUp`, `OnNetworkAsync`**.
- The author confirmed the game **allows mods to use networking functions** (his
  retrospective + the fact that PencilPal ships TCP code and was published).

### Transport used by PencilPal
- **GameMaker built-in TCP sockets** (not Steam):
  - Host: `network_create_server(network_socket_tcp, 6511, 1)`.
  - Client: `network_create_socket(network_socket_tcp)` +
    `network_connect(sock, ip, 6511)`; IP is typed by hand into a `get_string` dialog.
  - Async event `OnNetworkAsync`: `network_type_connect` / `network_type_disconnect` /
    `network_type_data` → `HandleNetworkingData(async_load)`.
  - Serialization: `buffer_create(1, buffer_grow, 1)` + `buffer_write(buf, buffer_u8/u64/...)`
    + `network_send_packet(socket, buf, buffer_tell(buf))`.
- Handshake: on connect the host pushes `SyncLand()`, `SyncResources()`, `updateVault()`.
- Ping/pong keepalive (first enum value is `PONG`).

### PencilPal's protocol (the Commands enum)
```
PONG, PLAYER_UPDATE, LAND, HP, DEL, VAULT, UPDATE, RESOURCE, DR, NPC, ENEMY, ITEM,
INDUST, DROID, LEVEL_UP, SEND_GOLD, CHAT, HOST_DISCONNECT, EMOJI, STARFALL, RANDOM_GEM,
CHEST, COOKIE, BATTLE, BATTLE_ACCEPT
```
Sync scope actually attempted: resources, islands/land tiles, breaking objects, dropped
items, object health, `image_index`, chests, industrial structures, NPC quests, slimes,
drones, vaults (a custom "Multiplayer Vault" structure), the partner puppet
(x/y/sprite/image_index/facing/angle/held item), gold transfers, chat, emojis, a paid
"store", and a PvP battle mode. Multi-zone handling via `objPlayer.room`/`hostRoom`.

### PencilPal's own open TODO list (lines 29–36 of Main.gml)
- Host is not synced with industrial structures.
- Land syncing is too intensive on big worlds; resource syncing needs work.
- Forge weapon upgrades not synced.
- "Changing menu crashing game" and "changing rooms deleting inventory" — both
  unreproducible, possibly side effects of `/recompile`.
- Emoji app not working; missing credits screen; missing a settings toggle for
  "sync on save".

## Modding API, from the official docs mirror (observed)

- **Live docs mirror:** https://topheranselmo.com/forager-modding-docs/ — a docsify SPA.
  Raw markdown is fetchable at e.g.
  `https://topheranselmo.com/forager-modding-docs/CoreFunctions/Modding/README.md`
  (served as `application/octet-stream`, so `webfetch` refuses it; plain HTTP fetch works).
  Sidebar (page list) at `.../_sidebar.md`. Original location was
  `http://138.197.196.157/docs/#/`. Written by **lazyeye**, built by **topherlicious**;
  current released API version **1.0.1**.
- **Mod structure:** every mod needs `fmConfig.json` + `Main.gml`. `Main` is required and
  runs at compile time; if it errors the mod is not activated. You **cannot create
  objects** — spawn blank instances (`ModObjectSpawn` / `ModObjectSpawnPersistent`) and
  attach methods with `InstanceAssignMethod`; wrap script references with `ScriptWrap`.
- **Lifecycle:** boot → intro → `rmInputSelect` → **`rmModCompile`** (first compile; also
  `/recompile` in the main menu) → `rmMainMenu` → `rmGame`. Hand-made locations
  (dungeons, space towers) live in their own rooms; the main world, Zones and the Void are
  in `rmGame`.
- **System step:** heavy work runs once per **20 frames**, spread across frames so one
  "slot" fires per frame; extend it via the `OnSystemStep` event. Includes pathfinding,
  cooldowns, structure crafting, dirt despawn, culling.
- **Culling / True Instance:** offscreen instances are **deactivated**; `ModObjectSpawn`
  instances are not culled, and `antiCull = true` opts an instance out. The game's
  **True Instance** system lets code reach deactivated instances — use `TrueInstanceFind`
  instead of `instance_find`. PencilPal's TODO ("Use TrueInstance") means it did not.
- **Controller objects:** `WorldControl` (main, persistent), `VoidControl`,
  `WeatherControl` (rmGame only), `TimeControl`, `LightControl`. Parents:
  `parSolid`, `parObject`, …
- **Functions seen (docs + source):** `ModObjectSpawn`, `ModObjectSpawnPersistent`,
  `ModNPCSpawn`, `ModEnemySpawn`, `ModStructureCreate`, `ModZoneCreate`,
  `InstanceAssignMethod` (events: draw/drawGUI/step/interact/clean up/destroy),
  `InstanceAddVariable` / `InstanceRemoveVariable` (serialization), `ScriptWrap`,
  `ScriptCall`, `ModFind`, `ModIsOnline`, `ModFunctionGet`, `ModGlobalGet`, `CommandCreate`
  (custom `/commands`), `ItemEdit`, `ItemCount`, `RemoveItem`, `HasSkill`,
  `WaterCollision`, `LighthouseCalculate`, `variable_global_get` / `_set` (extended GML),
  `ModSaveDataSubmit` / `ModSaveDataFetch`, `Trace`.
- **Events (`#define OnX(...)`)** include `OnLoad`, `OnSystemStep`, `OnStructureSpawn`,
  `OnChatMessage`, `OnLevelUp`, `OnStep`, `OnNetworkAsync`, `OnDrawEnd`, `OnDrawGUIEnd`.
  Events pass arguments from the game (e.g. `OnStructureSpawn(id, structure)`).
- **Tooling (official):** the *Forager Modding Tool* (create/upload mods, `.zip` on the
  docs site) and a *Forager API dialect pack* for **GMEdit Beta**
  (`assets/files/forager.zip`). Not downloaded yet; download only with owner approval,
  pinned by SHA-256 (rule 8).

### Community mod repos (examples of real, buildable mods)
- `Zamiell/FastCrafting` — https://github.com/Zamiell/FastCrafting — **GPL-3.0**, minimal
  (`fmConfig.json` + `Main.gml`). Shows `ItemEdit(Item.X, ItemData.CraftingTime, speed)`.
- `CainDev/Forager-Mods` — https://github.com/CainDev/Forager-Mods — source of several mods.
- `AlejandroAkbal/Forager-Cheater-Mod` — https://github.com/AlejandroAkbal/Forager-Cheater-Mod
- `Forager Evolved` — the first official mod pack (Workshop item 2147146084).

## Derived (our reading, not facts)

- A Forager co-op mod is a **pure GML** problem. `UndertaleModTool` / `GMEdit` operate on
  `data.win` and are only needed if the modding API is insufficient; start with the API.
- PencilPal is best described as **peer-simulated diff broadcast**: both clients run the
  full sim and ship diffs, so any missed/duplicated mutation desyncs or crashes. This is
  the likely root cause of the reported instability, not just bugs in individual packets.
- A from-zero design should therefore be **host-authoritative with sequenced mutations**
  (host resolves world changes, client sends *intents*, not results) and should tolerate
  lost/duplicate packets. That is exactly what the cancelled official netcode and
  PencilPal both lacked.
- Transport upgrade path that matches the hub's standard: keep the GML sync layer, replace
  raw TCP + typed IP with **Steam relay via Spacewar (appid 480)** as described in
  `playbook/networking-steam-spacewar.md`, so players never forward port 6511.
- Because Forager has no local multiplayer, the honest cheap answer *today* is still
  streaming (Steam Remote Play Together / Parsec). The build-from-zero path is a real
  project, sized in weeks, with the sync design above as the differentiator.

## Unknowns

- `unknown` — the current Steam build id of the live branch (owner copy's manifest said
  6160665; see `local/owner-state.md`). Whether the live build differs from the 2022/23
  one PencilPal targeted.
- `unknown` — whether PencilPal's source is available outside the compiled Workshop copy,
  and under what license. **We must not copy its code** without a license; study only.
- **Resolved 2026-10-19:** modding API docs mirror found —
  https://topheranselmo.com/forager-modding-docs/ (API version 1.0.1).
- `unknown` — whether Forager can run **two instances on one PC** (needed for automated
  two-client QA; may need `-nomutex` or a second Windows user).
- `unknown` — whether the modding API can load a native/GML extension
  (needed for Steamworks.gml inside a mod).
- `unknown` — whether `data.win` decompiles cleanly with UndertaleModTool for the live build.
- `unknown` — whether Foragipelago's source is public (would be a second, independent
  example of Forager mods doing real socket I/O).
- `unknown` — `data/directory.yaml` exact QA wording; **resolved:** Forager has no directory
  entry yet, and `python` is **not installed** (only the Windows Store stub, exit 9009), so
  `tools/build_directory.py` cannot run without installing Python (needs owner approval).

## Sources index

- yal.cc (YellowAfterlife, modding API + networking author): 2019/2020 reviews,
  `forager-retrospect`, `introducing-nuclear-throne-together`,
  `introducing-caveblazers-together`, `docs/gm/gmtogether`.
- Official modding API docs: https://topheranselmo.com/forager-modding-docs/
  (mirror; original `http://138.197.196.157/docs/`).
- PencilPal Workshop page (id 2843162754) and its source in the Workshop cache.
- PencilPal/PenPal guide: https://playco-opgame.com/multiplayer-mod/forager-multiplayer-mod/
- Press on the cancellation: RPS, TechRaptor.
- Community mods: Zamiell/FastCrafting, CainDev/Forager-Mods, AlejandroAkbal/Forager-Cheater-Mod,
  Forager Evolved (Workshop 2147146084).

## Decisions

- 2026-10-19: Path 2 (build from zero) chosen by owner.
- 2026-10-19: Treated the owner's non-Steam repack as unusable for identity/tests; asked
  the owner to restore the legit Steam copy first (details in `local/owner-state.md`).
- 2026-10-19: Dismissed PencilPal-as-base and GMTogether; kept PencilPal's *source* as a
  reference to study (no code copied).
