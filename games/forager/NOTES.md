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
| Steam build | `6160665` (Steam `StateFlags` 4; verified 2026-10-19) |
| `Forager.exe` SHA-256 | `AD2B90F164320F646C340B0BE60AF966851DFDC74DCE366580F3DCD531853AE6` |
| `data.win` SHA-256 | `0F5CCBFBBD80DD7EB5F1C4FDE676C4C1F8F1DC77E24BCCE9086D8EC2B38144E8` |
| `steam_api.dll` SHA-256 | `AF17DF745250D1814EAA274FFF7B0FAEB43381E6762E026267E5859778477ABD` |
| Caveat | **Modding is not Unity.** No BepInEx / Harmony / Il2Cpp. See "How modding works". |

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

## Verified in-game (2026-10-19, M0)

The scaffolding mod (`fmConfig.json` + `Main.gml`, a few globals and one `Trace` inside
`#define Main`) **compiled and loaded** in the live Steam build 6160665 through the game's
own loader — main menu → modding → enable → COMPILE. Console output (`t`), owner's SteamID
redacted:

```
Forager Co-op 0.0.1-m0 loaded.
Main method ran succesfully. Registering...
Generated ID for mod 'Forager Co-op' is '<steamid64>ForagerCoop'
Cleaning...
Finished loading and initializing mods!
Mods active: 1
Importing currently active mods to loader...
Scanning /mods/ to find inactive mods...
Found an inactive mod: PencilPal
```

What this pins down:

- **The toolchain works with no external tool.** No *Modding Tool*, no *GMEdit*, no
  `data.win` editing: a folder dropped into the mods directory and COMPILE in the game is
  enough to build and run GML. `Trace` prints to the `t` console.
- **Mod id rule:** the loader derives the id as `author` + `name` with non-alphanumeric
  characters removed — `<steamid64>` + `Forager Co-op` → `<steamid64>ForagerCoop`. So the
  `author` field in `fmConfig.json` is not cosmetic (it must be the SteamID64).
- **Mods directory** on Windows: `%LOCALAPPDATA%\Forager\mods\<Mod Name>\`. The enabled set
  and order live in `%LOCALAPPDATA%\Forager\mods\settings.json`, rewritten by the game to
  `{ "loadOrder": [ "mods/Forager Coop" ] }`.
- **The loader also scans subscribed Steam Workshop content** for the app, not only the mods
  folder — the log's "inactive mod PencilPal" is Workshop item `2843162754` sitting in the
  Workshop cache. So a co-op mod can be delivered either locally or through the Workshop.
- `Main method ran succesfully` precedes `Registering`, i.e. mod code really executes during
  the compile/registration pass.

### M1 network layer (2026-10-19/20)

The mod's M1 layer (HELLO/WELCOME handshake, PING/PONG, POS presence packets over DirectTCP
6511) was run against a **non-GML** peer (a Python script, now kept with the mod:
https://github.com/Egolz/forager-coop → `tools/coop_mockpeer.py`)
with the game hosting. The mod completed the GameMaker socket handshake, exchanged
HELLO→WELCOME (the peer saw the game's real mod id/version `0.0.2-m1`, protocol 1), and
ping/ponged both ways for ~25 s with no disconnects. The owner also confirmed the
disconnect path: the host logs `socket disconnected` + `still hosting` and keeps listening.

**Presence verified with real game data.** Standing inside a world, the mod sent 467 `POS`
packets in 26 s — `x=672 y=672 sprite=0 image=0..5 room=8` (the `image` field animating, i.e.
genuine per-frame state) — and, from the mock's `POS` packets, drew the remote marker; the
owner confirmed both the marker and the F8 status line. **Room 8 is the main world** (the same
constant PencilPal hard-codes as "host room"), which is useful for the M2 zone checks.

**Forager's mod sockets use GameMaker's own TCP protocol** — the part that bites anyone
writing a peer in another language:

- on connect the server sends the ASCII banner `GM:Studio-Connect\0` (18 bytes);
- the client answers with three little-endian u32s: `0xCAFEBABE`, `0xDEADB00B`, `16`;
- the server replies `0xDEAFBEAD`, `0xF00DBEEB`, `12`;
- every packet after that is framed `u32 0xDEADC0DE`, `u32 12`, `u32 payloadLen`, payload.

Constants confirmed against YoYoGames' own implementation
(`GameMaker-HTML5/scripts/functions/Function_Networking.js`) and verified live by dumping
the running game's bytes (the banner, then the step-3 signatures after our 16-byte reply).
Consequences:

- A relay/server written in another language must implement this handshake **and** the
  framing; a GML↔GML path (e.g. the Steam relay) gets it for free.
- GML also has `network_create_server_raw` / `network_connect_raw` for the unframed variant;
  whether Forager's API exposes them is `unknown` — the mod does not need them, since it
  talks to other GameMaker peers.

**Two instances on one PC: not possible** (owner-checked 2026-10-20 — launching Forager a
second time while it runs is refused). So automated two-client QA needs a second Windows
session/VM, or a second machine. Meanwhile the transport half can be tested with the
non-GML mock peer above.

**Terrain note for QA:** the mock peer also sends POS packets, so the game side of presence
can be inspected with the in-game status dump (F8 → Trace) without a second Forager copy.

### M2.1 host-authoritative structure placement (2026-10-20)

**Verified in the live build (2026-10-20):** the mod compiles and loads, F5 hosts on 6511,
and a mock peer completes GameMaker's socket handshake, `HELLO`/`WELCOME` (protocol 2),
`PING`/`PONG`, `POS` and the host's `STRUCTURE_TABLE` - **64 structures**, so
`Structure.Count` and `StructureGet(id, StructureData.Name/Object)` do work at runtime in
build 6160665 (see the caveat below about what the names look like).

**The round trip is verified live (2026-10-20, 01:58:39).** The mock sent
`INTENT_PLACE 'Лесопилка' at 809,651 (room 8)` and the host answered
`MUTATION_PLACE seq=1 intent=#15683 room=8 depth 0` on the console, with no crash and a live
socket. What is **not** verified - and is the open bug - is that the host *applied* it: the
player sees nothing appear in the world. The host's own apply path is therefore wrong
somewhere (`CoopCreateStructure` in `Net.gml`), and the next step is to probe it rather than
guess: F10 dumps the game's own structure instances next to ours (see the diagnostics note
below).

**Structure names look localized.** The 64 names the host sent were not ASCII on a Russian
install (the console mangles them), which would make name-keyed sync language-dependent -
the `STRUCTURE_TABLE` divergence check exists for exactly that. The peer now dumps the
table to UTF-8 JSON (`%TEMP%\coop_structure_table.json`) so this can be settled from the
file rather than the console. `unknown` until that file is read.

**Settled: the names are display strings, and the key has to change.** The dumped
`%TEMP%\coop_structure_table.json` holds 64 **Russian** strings, several with an embedded
newline (`"швейная\nмастерская"`, `"для\nпромышленности"`), i.e. wrapped UI labels, not
stable ids. The mock's English `PREFERRED_STRUCTURES` silently fell back to entry 0 for the
same reason. The right key is visible in the API docs' own event example:
`#define OnStructureSpawn(id, structure)` + `if (structure == Structure.Forge)` - the API
exposes a **`Structure` enum** with symbolic members, and that index is what PencilPal
actually syncs. `STRUCTURE_TABLE`/`INTENT_PLACE` should carry the index (name kept for logs).
Fix not yet made.

**F10 world probe (2026-10-20, 02:33): the structure was invisible because it was never a
structure.** The mock's request landed on structure **#0**, which this database exposes as
the build-menu **category header** `"для промышленности"` (`StructureGet(0,
StructureData.Object)` = `objEmpty`, the placement-preview placeholder). `objEmpty` is not a
`parStructure` descendant, so `instance_create_depth(..., objEmpty)` puts nothing real in the
world, and the preview ghost is gone by the next probe. Two more facts fell out of the same
dump:

- The database has **92 rows**, not 64: a hard cap in `CoopSendStructureTable()` cut the
  table at 64, so a peer only ever saw part of it. Rows #68..#80 have **empty names**
  (objects `objGate`/`objFence`/`objVoidPortal`/`objLure`) and cannot be addressed on a
  name-keyed wire at all.
- **Depth is `-y`.** The probe found `objPlayer` at `691,656 depth -656` and a hand-built
  `objFurnace` at `624,656 depth -656`, while the host had announced that same furnace as
  `depth 0`: `OnStructureSpawn()` fires *before* the game y-sorts the instance. We were
  creating remote structures at depth 0, i.e. on the wrong layer.

Fix (mod `0.0.5-m2c`, protocol unchanged at 3): a new `CoopStructDepth(y)` returns `-floor(y)`
and is used by both the applying host and the receiving client, so the mutation's `depth`
field is now informational. `CoopSendStructureTable()` lost its 64 cap and skips unnamed
rows. The mock's `choose_structure()` skips the `"для…"` headers and gained `--place-index`,
so its automatic choice cannot land on a placeholder again.

**Anomaly, not yet explained:** in rows #46..#80 the name column and the object column look
**misaligned** - `"каменная стена"` (stone wall) reads back as `objGate`, `"железная стена"`
as `objEmpty`, cycling `objEmpty`/`objGate`/`objFence` with period 3 while the names
alternate gate/wall with period 2, and #68..#80 have no name at all. Rows #1..#45 and
#81..#91 are correct and coherent (#1 `"печь"` = `objFurnace`, the object the game really
used for a hand-built furnace), so the fence/gate family is a separate problem:
`StructureData.Object` may not be the object index for those rows. Flagged for M2.x; it does
not touch ordinary buildings.

**A mod here cannot write a file.** Live COMPILE error, 2026-10-20:

```
Mod failed to unit! Forager Coop/Net.gml[L453,c1] `ini_open` is not function or script
```

`ini_open`, `ini_write_string`, `ini_write_real` and `ini_close` are all *in* `data.win` -
the game writes its own `secretSettings.ini` with them - so "the name is in the build" is
necessary but **not sufficient**: the loader resolves a mod's calls against its own function
table. `file_text_*`, `buffer_save`, `show_debug_message` and `working_directory` are absent
from the build entirely. `tools/check_gml.py` now has a `LOADER_BLOCKED` list for this class
(it is exactly the false "0 problems" that shipped the broken build). Consequence: a long
dump cannot become a file - the mod forwards it to the peer over the protocol
(`DEBUG_LINE`, packet 12, protocol 3) and `tools/coop_mockpeer.py` writes
`%TEMP%\coop_debug.txt`. Offline `tools/mock_e2e.ps1` covers that path too.

**More from the official docs mirror** (API 1.0.1, fetched 2026-10-20):

- `parStructure [parObject]` is the parent of **every** craftable structure, "both from the
  build menu and from placing objects from the inventory (like decorative items)".
- Structure **crafting runs on the system step** (once per 20 frames, with pathfinding and
  culling) - so a structure is not necessarily a thing the game builds on the frame it is
  placed.
- Culling deactivates off-screen instances; `antiCull = true` in a Create event prevents it,
  and `TrueInstance*` is the documented way to *reach* deactivated instances. Our probe uses
  `TrueInstance*`, so an instance that exists but is culled/deactivated (invisible to the
  player, present to the mod) is exactly what it can distinguish - one of the live theories
  for the invisible structure.
- Mods are expected to create their own instances with `ModObjectSpawn(x, y, depth)`;
  "you cannot create objects in Forager modding". `instance_create_depth` with a *native*
  object is still used in the docs' own examples (and by PencilPal), so creating a native
  structure object is legitimate - the question is which object and with what initial state.
- The API docs **do not document** `StructureGet` / `StructureCreate` / `StructureEdit` /
  `StructureAddItem`: the `Structure Database` page only links a tutorial video
  (YouTube `DMn9r4wmJFo`). For structure internals we are back to PencilPal's source plus
  live probing.

The notes below are what was learned while writing it (from the modding API docs,
PencilPal's source and the live build's symbol table).

**The structure database is readable at runtime.** For an id in `0 .. Structure.Count-1`:

- `StructureGet(id, StructureData.Name)` → the structure's key, **lowercase** (`"forge"`,
  matching what PencilPal compares: `"multiplayer vault"`).
- `StructureGet(id, StructureData.Object)` → the object index it is built from (`objForge`).

A received structure is created with `instance_create_depth(x, y, depth, object)` and then
`structure = <structure id>` set on that instance - exactly what PencilPal's receive path
does (`Networking.gml:937`: `with(instance_create_depth(_x, _y, _depth, _item)) { structure = _struct; }`)
and what `CoopCreateStructure` does. PencilPal also sets a `global.deny` flag around the
creation so the mod's own `OnStructureSpawn` does not echo it back, which is the same guard
as this mod's `global.coopApplyingRemote`.

`Structure.Count` is the one call here not yet exercised live: PencilPal never uses it (it
syncs **object indices**, so it needs no id/name table) and it comes from the API docs. If
it does not exist at runtime, `error.log` names the line, and the fallback is to identify
structures by object name (`object_get_name(i.object_index)` / `asset_get_index(name)`)
instead of building the database at all - good enough for base-game structures, though not
for mod structures, which all share `objModStructure`.

`StructureData` / `StructureType` are **compile-time enums**, so they do not appear in
`data.win`'s symbol table even though the code above works — an absent name there is not
proof of an absent feature. The database is not guaranteed to be up when a mod's `Main`
runs, so the mod caches it lazily (rebuild if the cache is empty) instead of once at load.

**`#define` macros here behave like functions, not like textual includes.** Evidence:
PencilPal's `EaseOutBack` is used mid-expression (`var _y = EaseOutBack(_t, a, b, d);`) and
its body ends in `return` — as a text include that would be a syntax error, and the early
`return` would abort the caller's script (their phone UI would never draw). This matters
because it means (a) `return` in a macro returns from the macro, (b) macros may be nested
freely, and (c) re-declaring the same `var` name in different macros is harmless. PencilPal
also shows the game's compiler tolerates the same `var` name twice in *different blocks* of
one script (`Battle`, `SyncResources`), which is why 20 `var _packet` declarations coexist in
`Networking.gml`.

**Structures travel by name, not object index.** An object index is build-specific and
unreadable in a log; the name is neither. A build that renumbers structures keeps working,
and a log line says `place 'forge'` instead of `place 214`.

**Validation is the host's job.** The host refuses an intent unless the structure exists, the
client's reported room equals the host's room, and the target is within 448 px of the
client's *last reported position* (a sanity check against a peer that teleports builds
across the map — not anti-cheat). A refusal is answered with the reason, which the client
`Trace`s, and the client rolls its optimistic placement back.

**Protocol 2** — the packet set changed, so `coopProtocol` and the mock peer went 1 → 2;
a v1 peer is now rejected at handshake instead of silently ignoring the new packet ids.

**How this loader fails, learned the hard way (verified in the live build, 2026-10-20).**
The first M2.1 deploy did nothing at all in game: no error on screen, no console line, F5
dead. The loader's own console did say why:

```
Loading file: mods/Forager Coop/Net.gml...
Mod failed to unit! Forager Coop/Net.gml[L137,c26]: Expected a statement, got number
Exiting compile...
```

Several facts fall out of that, all worth knowing before writing any Forager mod:

- **There are no value macros.** `#define COOP_PLACE_RANGE 448` is rejected: this loader
  reads a macro body as *statements*, so a bare number is "Expected a statement, got
  number" and the whole mod refuses to load. A constant has to be a global (M1 happened to
  use none, which is why M1 loaded). Note the typo in the loader's message ("failed to
  unit") if you ever grep for it.
- **Not every GML builtin exists in this build.** `array_delete()` and
  `variable_instance_exists()` are absent from the 107 MB `data.win`'s symbol table while
  `array_get`, `array_push`, `array_length` and `instance_number` are present. Which
  functions a build has is answerable offline by searching `data.win` for the name — and
  that is now the mod's pre-deploy check (`tools/check_gml.py`, which also flags brackets,
  duplicate `var`s in one block, wrong macro arity and the value-macro trap). A mod compile
  itself reports only the *first* error, so an offline pass saves a round trip each time.
  **Caveat, learned later the same day:** presence in `data.win` is necessary but not
  sufficient — the string search says yes to `ini_open`, and the loader still refuses it
  (see the file-writing note below and `LOADER_BLOCKED` in the checker).
- **`try`/`catch` does not compile.** The loader turns mod GML into its own AST and cannot
  walk a `try_catch` node, so a `try` anywhere takes the game down at COMPILE time - and
  again on every launch, because mods are recompiled. There is no on-screen message and the
  loader console just stops (`Exiting compile...`); the game's own
  `%LOCALAPPDATA%\Forager\error.log` is the only place that names the culprit:

  ```
  ERROR in Other Event: User Defined 0 for object ModControl:
  Can't seekAllOut over Forager Coop/Net.gml[L150,c1] try_catch
  ```

  That is how the *second* M2.1 deploy died, after the value-macro fix above, and why the
  mod was then rewritten without a single try/catch (guards are plain `if`s). It also
  explains PencilPal's standing TODO *"Fix recompile crashing game"*: that mod uses
  `try/catch` in `Networking.gml` and `GUI.gml`, so it cannot be recompiled at all in this
  build.
- **`error.log` is the oracle.** `%LOCALAPPDATA%\Forager\error.log` records both compile and
  runtime failures, with a file and line for mod code. A mod that "does nothing", or a game
  that dies on COMPILE, should be diagnosed from there before anything else is changed.
- **The two failure modes differ.** A parse error the compiler *understands* is reported
  nicely in the loader console (`Mod failed to unit! <file>[L,c]: <reason>`) and the game
  survives it. A construct it cannot handle at all (`try_catch`) crashes the game instead,
  with only `error.log` to say why. Both were hit in one session on this game.

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

- **Resolved 2026-10-19:** live Steam build is **6160665**, verified by Steam "Verify
  integrity" (21 files, all valid) on the owner's machine. `Forager.exe` and `data.win`
  SHA-256 are in the Identity table. Whether this equals the 2022/23 build PencilPal
  targeted is still `unknown` (PencilPal's manifest only records API version 1.0.1).
- `unknown` — whether PencilPal's source is available outside the compiled Workshop copy,
  and under what license. **We must not copy its code** without a license; study only.
- **Resolved 2026-10-19:** modding API docs mirror found —
  https://topheranselmo.com/forager-modding-docs/ (API version 1.0.1).
- **Resolved 2026-10-20:** Forager **cannot** run two instances on one PC — a second launch
  while the game is running is refused. Two-client QA therefore needs a second Windows
  session/VM or a second machine.
- `unknown` — whether the modding API can load a native/GML extension
  (needed for Steamworks.gml inside a mod).
- `unknown` — whether `data.win` decompiles cleanly with UndertaleModTool for the live build.
- `unknown` — whether Foragipelago's source is public (would be a second, independent
  example of Forager mods doing real socket I/O).
- `unknown` — whether PencilPal's `Networking.gml` byte layout could be reused as-is, or
  whether the sequenced-mutation design needs a different packet schema (probably the
  latter).
- **Resolved 2026-10-19:** Forager had no directory entry; one is added in this PR, and
  Python 3.12 + PyYAML are installed so `tools/build_directory.py` runs.

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
- 2026-10-19: **M0 passed** — the scaffold mod compiles and loads in the live build using the
  game's own loader, so no external modding tool is needed to build co-op. Next is M1
  (network layer) per `DESIGN.md`.
