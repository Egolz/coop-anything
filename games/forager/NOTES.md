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

**The round trip is verified end to end (2026-10-20, 02:46).** The mock asked for `"печь"`
at 800,672, the host answered `MUTATION seq=1 … depth -672` and then re-announced the same
instance itself as `seq=2`, and the next F10 probe found it as a real instance:
`par objFurnace 800,672 depth -672 image 0 visible 1`, counted under both `#1 'печь'` and
`#81 'ядерная печь'` (same object, two database rows). `depth = -y` is confirmed against the
game's own instance, and the table now carries **79 addressable** entries (92 rows minus the
13 unnamed ones).

**One behavioural gap found by the same test, since closed:** the mock's guessed spot was on
**water**, and the furnace was created anyway. A client's intent was validated for *who* asked
(the name, the room, the 448 px range) but **not for whether the spot is legal**. `0.0.7-m2e`
adds the game's own check - see the `WaterCollision` finding below.

**`ScriptCall` takes exactly one argument (live COMPILE, 2026-10-20):**

```
Mod failed to unit! Forager Coop/Net.gml[L536,c16] `ScriptCall` takes 1 argument, got 3.
```

The docs read as if a wrapper is only *described* by `[mod, methodName]`, but the arity error
is the ground truth: `ScriptCall(wrapper)` passes the wrapper and nothing else. So
`ScriptWrap` + `ScriptCall` reaches a **zero-argument** first-party script only, and
`WaterCollision(x, y)` - which needs x and y - is not reachable through that path. `F11` is
therefore a read-only probe (`0.0.6-m2d`) that reports `typeof`/`array_length` of the wrapper,
what `ModFunctionGet("Forager", "WaterCollision")` returns, and a `GetLandsOwned` control call,
instead of guessing at an argument-passing spelling. How (or whether) arguments can be passed to
a first-party API script is `unknown`; the API symbol table also holds `ModCall` and `InsideMod`,
untried.

**What a `ScriptWrapper` actually is (F11, live 2026-10-20):** not the documented
`[mod, methodName]` pair but a **3-element array**

```
ScriptWrap type   = array, len 3
  wrap[0] = wrapper
  wrap[1] = EgolzForagerCoop
  wrap[2] = WaterCollision
```

The tag `"wrapper"` comes first and the mod id second. So `ScriptWrap("WaterCollision")` does
not reach the game's own script at all - it resolves the bare name **inside our own mod** and
hands back a lazy reference to it, which is why
`ScriptCall(ScriptWrap("WaterCollision"))` could never have asked the game about water.
`ModFunctionGet("Forager", "WaterCollision")` returns `undefined`, so the first-party id is
**not** `"Forager"`; whatever it really is has to be discovered (`ModFind` / `ModIsOnline`, now
part of the probe) rather than assumed. The one control that did run,
`ScriptCall(ScriptWrap("GetLandsOwned"))`, returned `-1` - consistent with "no lands owned", but
also with "the name resolved inside our mod and hit nothing", so it proves the *call path* works
and not that first-party scripts are reachable.

**Settled - and the answer was "just call it" (live 2026-10-20, `0.0.7-m2e`):** the game is
**not registered as a mod at all**. `ModFind` returned `-1` and `ModIsOnline` `0` for every id
tried (`Forager`, `ForagerModdingAPI`, `ModdingSample`, `forager`, `Forager Coop`), so the
registry only ever holds real mods and `ModFunctionGet("Forager", ...)` could never have worked.
But the functions the docs list under *General Forager Behavior* are **directly callable** - a
bare `WaterCollision(x, y)` compiles and runs:

```
WaterCollision(703,630) = 0     (the player stood on land)
left 0   right 1   up 1   down 0   far +512 right 1
HasSkill(0) = 0
```

So `0` = land, non-zero = water, and no wrapper is involved. The F11 probe, the `ScriptWrap` /
`ModFunctionGet` detour and both around it were unnecessary - the docs' plain list was the whole
story. **Lesson for the next agent:** try the plain call first; reach for `ScriptWrap` /
`ScriptCall` only for a *mod's own* scripts, and remember `ScriptCall` takes the wrapper alone.

The host now refuses an intent whose spot is water, and the round trip is verified:

```
-> INTENT_PLACE 'печь' at 859,641 (room 8, #2268); our puppet is at 635,649
<- INTENT_REJECTED #2268: 'that spot is on water'
```

The client rolls its optimistic placement back on `INTENT_REJECTED` (it had already drawn the
furnace optimistically), so both ends end up agreeing with no structure on the water.

**A refusal alone is not proof, so the mock now supplies the other half (2026-10-20).** The
host's spot check is only shown to work if the *same* run also places something the host
accepts. The mock has no `WaterCollision` of its own, so it learns where the land is by trying:
after a `"that spot is on water"` refusal it walks a ring of offsets around the game's player
(all within the host's 448 px range) and stops at the first confirmed `MUTATION_PLACE`. A refusal
for any other reason stops the walk instead of repeating it, so the reason is the finding. The
state machine is covered offline by the selftest (a socketpair, no game):

```
selftest: 11 packets encode/parse correctly, spot-retry ok
-> INTENT_PLACE 'forge' at 728,600 (spot 1/10)
<- INTENT_REJECTED #9839: 'that spot is on water'
   retrying at another spot (water was at (728, 600))
-> INTENT_PLACE 'forge' at 472,600 (spot 2/10)
<- MUTATION seq=7 intent=#7576 room=8 'forge' at 472,600 depth -600
   accepted: ... the spot we asked for ((472, 600)) was land
```

`tools/mock_e2e.ps1` still passes (`handshake, welcome, pong, structure table and INTENT_PLACE
all seen`).

**And the round trip, live 2026-10-20 (`0.0.7-m2e`), refusal and acceptance in one run:**

```
[03:20:25] -> INTENT_PLACE 'печь' at 853,660 (room 8, #29143, spot 1/10)
[03:20:25] <- INTENT_REJECTED #29143: 'that spot is on water'
[03:20:25]    retrying at another spot (water was at (853, 660))
[03:20:25] -> INTENT_PLACE 'печь' at 594,654 (room 8, #7980, spot 2/10)
[03:20:25] <- MUTATION seq=1 intent=#7980 room=8 'печь' at 594,654 depth -654
[03:20:25]    accepted: ... the spot we asked for ((594, 654)) was land
[03:20:25] <- MUTATION seq=2 intent=#0 room=8 'печь' at 594,654 depth -654
```

`seq=2` is the game's own `OnStructureSpawn` re-announcing the instance it just made, exactly as
in the earlier M2.1 run. So the host's spot check is verified in **both** directions.

**Two known gaps in the placement rules, raised by the owner after this run - one confirmed, one
open. Neither is part of M2.1's wire work:**

1. *A structure cannot stand on an occupied tile* (another heater, a tree, a rock). The host
   currently checks **water only**. Nothing stops an intent for a spot the game would refuse, so
   this is a real hole, not a hypothetical. Open question: is there a first-party predicate for
   it? `WaterCollision` was luck - it is documented under *General Forager Behavior* and callable
   directly. The tile/occupancy side shows `objTile`, `parSolid` (the parent of solid objects),
   `objWateredTile`, and internal scripts `TileCollision`, `GridCollision`, `CheckTilemap`,
   `CheckDirtRect` - **but the docs list no plain predicate** for "is this cell free". Trying
   internal script names blind is expensive (a wrong name fails the whole COMPILE). Status:
   `unknown`, needs a probe, not a guess.
2. *A structure costs resources.* Right now `INTENT_PLACE` **creates the structure without
   spending anything** - the host applies it directly, out of band from any inventory. So a
   joining player builds for free, which is not acceptable in a real session. This cannot be
   fixed before **the client's inventory is synchronised** (M2.2/M2.4): there is nothing yet to
   charge. Ordering is therefore M2.2 (loot/resources) -> then a costed placement that the host
   validates against the client's synced inventory.

**Occupancy, first step: F12 (read-only probe), `0.0.8-m2f`.** The owner asked whether a
structure can be placed on top of another structure, a tree or a rock, and the honest answer
today is **yes it can - the host checks water only**. The documented modding API does **not**
expose an occupancy predicate: `CoreFunctions/GeneralForagerBehavior` lists exactly three
callable functions (`HasSkill`, `LighthouseCalculate`, `WaterCollision`), and the
`CoreFunctions/Modding` reference has no "is this cell free" entry. So `WaterCollision` was
lucky rather than representative - it happens to be documented *and* directly callable.

F12 therefore **measures** the spot with calls already proven live in F10, and does not guess at
a name:

- it resolves `parStructure`, `parResource`, `parSolid`, `objTile`, `objTree`, `objRock`,
  `objBush`, `objCactus` by name and prints each one's `instance_number` (a missing parent is
  itself a finding - a check that measures nothing looks like "free");
- it counts instances of those parents within **24 px** of the player, which is the build-cell
  size, and prints each hit with its object name and distance.

The **collision builtins** (`instance_position`, `place_meeting`, `position_meeting`,
`place_free`) are the *right* answer and are all in `data.win` - but being in the build is
necessary, not sufficient (`ini_open` was in the build and the loader still refused it), so they
stay commented in F12 and get **one dedicated COMPILE** each, exactly like the bare
`WaterCollision` call did. `data.win` also has `tile_get_index`, `tilemap_get_at_pixel` and
`layer_tilemap_get_id` at count **0** - the tilemap accessors are *not* in this build, so a
tile-map-based reading is off the table and the parent-object route is the one worth taking.

**F12's second live run settled the rule's shape and the rule is now implemented (`0.0.10-m2h`).**
The parent chains, walked with `object_get_parent` (already proven live in F10), are the whole
story:

```
chain objFurnace   -> objFurnace  parStructure  parObject  parSolid
chain objTree      -> objTree     parNatural    parObject  parSolid
chain objRock      -> objRock     parNatural    parObject  parSolid
chain parStructure -> parStructure parObject     parSolid
```

**Every tree, rock and built structure descends from `parSolid`** - a furnace through
`parStructure`, a tree or rock through `parNatural`, all meeting at `parSolid`. So a single rule
suffices; a rule against `parStructure` alone would have missed trees and rocks entirely, and
would also have looked broken, because `parStructure` reports **0 instances of its own** (the
game keeps a furnace under `objFurnace`). The same run confirmed the reading on a real object:

```
parent 'parStructure' #108 instances 1        (the furnace that was just built)
parent 'parSolid'     #271 instances 20       (19 world solids + the furnace)
  near objFurnace at 720,624 d 13 (parent parSolid)
```

`CoopSolidNear(x, y)` in `Net.gml` counts `parSolid` instances within `global.coopSolidRadius`
(24 px - the live reading put a tree at 11 px and a rock at 17 px, so 24 px is the build-cell),
using only calls already proven live. The intent handler refuses `"that spot is already
occupied"` after the water check. The mock treats an occupied refusal like a water one and steps
to the next candidate, so a run shows both obstacles being walked around.

**Verified live 2026-10-20, all four cases in one afternoon.** The mock was given a second round
that aims at the exact cell it just filled, so the occupied rule is answered by the host's own
new structure and not by luck:

```
[03:40:11] accepted: 'MUTATION_PLACE' at 665,578 (seq 1) - the spot we asked for was free
[03:40:11]    round 2: will ask for the occupied cell (665,578)
[03:40:14] -> INTENT_PLACE 'печь' at 665,578 (round 2, spot 1/1)
[03:40:14] <- INTENT_REJECTED #33770: 'that spot is already occupied'
```

and the owner standing next to a tree, where the first five candidate cells were all occupied:

```
[03:41:14] -> INTENT_PLACE 'печь' at 664,656 (round 1, spot 1/25)
[03:41:14] <- INTENT_REJECTED #64191: 'that spot is already occupied'
[03:41:14] -> INTENT_PLACE 'печь' at 680,656 (round 1, spot 2/25)
[03:41:14] <- INTENT_REJECTED #57968: 'that spot is already occupied'
... spots 3, 4, 5 the same ...
```

So the rule holds in all four states: **free -> accepted, water -> refused, occupied -> refused,
and a cell filled one second earlier -> refused**. The second run ended with every candidate cell
refused ("the ring is all water or occupied") because the owner was on a small island and the
ring reaches 320 px - a correct outcome, not a failure: the host refused everything illegal and
the mock had nowhere legal to ask. Worth noting for future tests: the candidate ring should
follow the *owner's* island, or the mock should ask at a cell it knows is free.

**Then the owner caught the gap that made the whole thing look wrong: a player is not a build
spot (`0.0.11-m2i`).** The dense spiral's first candidate was offset `(0,0)` - the cell the player
stands on - and the host **accepted** it, so the mock's furnace appeared *under the host*:

```
[03:47:29] -> INTENT_PLACE 'печь' at 595,656 (round 1, spot 1/169)
[03:47:29] <- MUTATION seq=1 intent=#22357 room=8 'печь' at 595,656 depth -656
[03:47:29]    accepted: ... the spot we asked for ((595, 656)) was free
```

`parSolid` does not cover characters, and in the real game you cannot build on yourself or on
another character - so the check was incomplete, not the rule. `CoopSolidNear` now also counts
`objPlayer`, `parNPC`, `parEnemy` and `objPet` inside the 24 px cell, resolved lazily and **skipped
when this build does not have them**, so a renamed object costs a missing guard and never a
COMPILE. `parEntity` is deliberately **not** in the guard: if it is a broad root (a parent of
`parSolid` too) it would refuse legal spots, and a check that refuses too much is worse than one
that misses a rare case - so F12 now dumps its chain and the next run adds it only if it really
is the living-things parent. The mock also no longer offers the player's own cell as a candidate,
which is the model the real game uses.

**Verified live after the fix (`0.0.11-m2i`), the full sequence in one run:**

```
[03:51:21] -> INTENT_PLACE 'печь' at 707,666 (round 1, spot 13/168)
[03:51:21] <- INTENT_REJECTED #35835: 'that spot is already occupied'
[03:51:21] -> INTENT_PLACE 'печь' at 723,602 (round 1, spot 14/168)
[03:51:21] <- MUTATION seq=1 intent=#8625 room=8 'печь' at 723,602 depth -602
[03:51:21]    accepted: ... the spot we asked for ((723, 602)) was free
[03:51:21]    round 2: will ask for the occupied cell (723,602)
[03:51:24] -> INTENT_PLACE 'печь' at 723,602 (round 2, spot 1/1)
[03:51:24] <- INTENT_REJECTED #55420: 'that spot is already occupied'
```

The owner confirmed the visual half too - *"the mock put the furnace on a free spot"*, on the
ground and **not** under the host. The spiral's first candidate is now `(-16,-16)`, so the cell
the player stands on is never offered, matching the real game.

**Anomaly, not yet explained:** in rows #46..#80 the name column and the object column look**misaligned**

**Anomaly, not yet explained:** in rows #46..#80 the name column and the object column look**misaligned** - `"каменная стена"` (stone wall) reads back as `objGate`, `"железная стена"`
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

### M2.2 item database (2026-10-20) - the F3 probe

Started M2.2 (resources and loot) by measuring the item half before writing any sync code, the
same order that found `WaterCollision` directly callable. Mod `0.0.12-m2j`, hotkey **F3**
(read-only).

**The event list came from PencilPal, not the docs.** The official docs document exactly one
mod event (`OnStructureSpawn`), and mod event names are **not in `data.win`** (`OnStructureSpawn`
appears 0 times there): the loader fires an event by looking for a file of that name in the
mod's `Events/` folder. So the authoritative list is a mod that uses them - PencilPal's
`Events/OnEvent.gml` holds the full set with real signatures:

| Event | Signature | What it gives us |
|---|---|---|
| `OnResourceSpawn(l)` | instance | a resource appeared (tree, rock, ...) |
| `OnResourceDestroy(i)` | instance | a resource was harvested |
| `OnItemDrop(d)` | array of `{x, y, item, stack, id}` | items fell into the world |
| `OnItemGet(items, size)` | item name, count | the player picked something up |
| `OnMobSpawn(inst)` / `OnMobDeath(inst)` | instance | enemies (PencilPal leaves these `return`ed - WIP) |
| `OnStructureInteract(type, structure, inst)` | ... | a structure was used |
| `OnPlayerDamage(dodge, damage)`, `OnLevelUp(level)`, `OnRoomLoad()`, `OnSave()`, `OnLoad` | ... | misc |

**The Item Database API** (documented, all five present in this build's symbol table - each
appears exactly once): `ItemGet(id, ItemData.Name)`, `ItemCount(id, vip=false)`, `GainItem(item,
qty)`, `RemoveItem(item, qty)`, `DropItem(x, y, item, qty)`. `Item.Count` is the row count.

**Live F3 result (owner's install, 2026-10-20):**

```
Item.Count = 319, cached 319
anchor Wood  = 0    name 'Древесина'   found-by-name -1   owned 0
anchor Stone = 1    name 'Камень'      found-by-name -1   owned 0
anchor Coin  = 111  name 'Монета'      found-by-name -1   owned 0
```

- **319 items**, and the anchors land exactly where the docs and PencilPal assume:
  `Item.Wood == 0`, `Item.Stone == 1`, `Item.Coin == 111`. So the ids are stable and the wire
  can key on them - which is what M2.2 needs.
- **The item names are localized, exactly like the structure names.**
  `ItemGet(Item.Wood, ItemData.Name)` returns `"Древесина"` on a Russian install, so
  `CoopFindItem("Wood")` returns `-1` and always will. This is the same lesson the structure
  table taught, one milestone later: **an id travels, a name is for the log.** The divergence
  check that the item table originally ran (count matching names) would therefore have
  screamed on two installs that merely run different languages while their ids are fine - it
  was rewritten to compare **row count + the anchor ids**, which are build constants and
  language-independent, and to use the names only to name the row that moved.

The item table travels as new packet **13 `ITEM_TABLE`** (`u16 count`, `{ u16 id, string name }`),
sent right after `WELCOME` alongside the structure table; the mock parses and logs it, its own
host sends one, the selftest round-trips it (12 packets) and the e2e run asserts it, so the
packet is covered offline before a game speaks it.

### M2.2 loot and resource sync (2026-10-20) - live, first harvest verified

Mod `0.0.13-m2k` (protocol **4**), then `0.0.14-m2l` for the duplicate investigation.
Three new packets and three new event handlers.

**First live harvest (2026-10-20, tree at 664,680, room 8).** Felling one tree delivered
the whole loop, every packet decoding correctly:

```
<- RESOURCE_DESTROY obj#85 at 664,680 room 8
<- DROP item #0 x1 at 664,680 room 8
<- DROP item #0 x1 at 664,680 room 8
<- PICKUP item #0 at 673,679 room 8
```

- `OnResourceDestroy` carries the **object index** (85), not a name - as designed.
- `OnItemDrop` carries **item #0** = Wood, the id F3 had pinned.
- `OnItemGet` carries **item #0** and *no tile*; the `PICKUP` position (673,679) is the
  **player's** position, and the drop was found next to it - which is the whole reason
  `CoopApplyPickup` searches by radius instead of trusting a coordinate the event never
  sent.

**The open question the live run created: why TWO identical DROPs.** Both were
byte-identical (`664,680`, item 0, stack 1). From the wire alone that is ambiguous:
one call carrying a two-entry array (two piles that really exist) reads exactly like the
event firing once per instance (one pile reported twice). The difference matters on the
receiver - one is a pile to mirror, the other a pile to dedupe. So `0.0.14-m2l` does two
things: `OnItemDrop` now traces the **array length** (which answers it on the next
harvest), and `CoopApplyDrop` gained a second dedup layer - a short-lived pending list for
the case where the duplicate arrives *before* `DropItem()` has materialised its instance,
which is the timing the live run actually showed. The list is drained each step by
`CoopTickPendingDrops()`, so it cannot grow without bound and a genuine later drop still
lands.

**The event signatures came from PencilPal, again.** The docs document **none** of these
three, and the names are absent from `data.win` (they are file/define names the loader
looks up), so the only source is a mod that runs them - `Events/OnEvent.gml`:

| Event | What it really carries |
|---|---|
| `OnItemDrop(d)` | `d` is an **array**, one entry per dropped item, each with `.x .y .item .stack .id` |
| `OnResourceDestroy(i)` | the instance `i`; used as `i.x`, `i.y`, `i.object_index` |
| `OnItemGet(items, size)` | `items` = an **item id**, `size` = the count. **No position at all** |

The `OnItemGet` shape is the one that changed a design. PencilPal's own handler
*reconstructs* the tile by walking `parResource` and matching a drop within 20-30 px of the
mouse, the player or a droid. So there is no tile in the event; our `CoopApplyPickup()`
does the same walk on the far side instead of pretending to know an x,y the wire never
carried. It searches `parResource` for a drop of that **item id** within ~48 px of the
position the picker reported (their own player).

**What a drop's tile is matched on** is `parResource`'s own `item` field - PencilPal reads
`parResource.item` directly (line 161 of its `OnEvent.gml`), so no `ItemGet` name
comparison is needed and no localized string is involved. That is also why `CoopDropAt()`
walks `TrueInstanceNumber(parResource)` / `TrueInstanceFind` rather than `instance_number`:
a culled (offscreen) drop still has to be found when the other player picks it up.

**One limit copied from PencilPal's own code, recorded rather than fixed:** its
`OnItemDrop` can `instance_destroy(d[0].id, false)` and `return` before forwarding (an
industrial building absorbing its output, or `global.land`). Our handler announces every
drop it is shown, so a drop the game swallows may briefly be announced. The next reconcile
(pickup/destroy) corrects it; this is the first thing to watch in the live run, not a claim
that it is right.

**Scope, on purpose:** this syncs the **world** only. A drop that leaves the ground leaves
it on both machines; **nothing moves between the two players' inventories** - `GainItem`/
`RemoveItem` are the inventory half and are their own step (M2.4). Doing both at once is how
the placement slice grew its "invisible structure" bug, so it is one claim at a time again.

Offline coverage: the mock encodes, parses, describes and round-trips packets 14-16; the
selftest is 15 packets; and the e2e session now has the mock host announce a `DROP`, a
`PICKUP` and a `RESOURCE_DESTROY` that the mock client decodes, so all three are exercised
with no game running.

**Two GML availability checks that came out of writing this** (offline, `data.win` string
counts): `is_struct` **0**, `variable_struct_exists` **0**, `variable_instance_exists` **0** -
so none of the "is this thing there" predicates exist to guard a struct field read, and the
field access is the only test. `is_array` **2**, `variable_instance_get` **2** and
`array_set` **1** do exist. This is the same trap as `ini_open`: a build's symbol table
decides what a mod may even *name*.

**A test-tool lesson that cost one run (2026-10-20).** The first live attempt of this build
failed with `REJECT reason='protocol mismatch: host 4, peer 3'` on every reconnect: the
freshly compiled game spoke protocol 4 while the **background mock was still the previous
process**, running the old constant. Bumping `PROTOCOL` in the source does nothing to a mock
that is already running. **Restart the mock after every protocol bump**, or the test lies -
the game is fine, the harness is stale.

**A citrus drop corrected the dedup design (2026-10-20).** The second live harvest showed
three DROPs at one tile: two wood (`#0`) and one citrus (`#55`). The owner explained the
citrus is a random bonus a tree can give alongside its wood, so a multi-item fire is normal -
and, crucially, **the game drops several wood stacks as separate piles**. That inverted the
earlier guess: "two equal items at one tile" is the *common* case here, not a duplicate, and
the receiver-side dedup (`CoopDropAt` by position+item) would have **silently swallowed a
player's second wood**. `CoopApplyDrop` no longer dedupes at all; it creates exactly what it is
told. The only duplicate distinguishable from the wire is the same instance `id` listed twice
in one fire, so that - and only that - is filtered, on the **sender**, where the drop's own
`id` is visible. Cross-fire duplication is left unhandled pending the `DEBUG_LINE` measurement
(an `OnItemDrop` diagnostic, since `Trace()` never leaves the machine).

### M3 blueprint (2026-10-20) - the visible second player, from working code

The owner asked to see the other player: model, name over the head, held item, cosmetics. This
is M3, and PencilPal - the one mod that runs in this build - **already draws exactly that**, so
the recipe is known rather than guessed. Its `Events/OnDrawEnd.gml` is the reference:

```gml
// the partner's held item, drawn beside them
var _sprite = ItemGet(global.partnerHold, ItemData.Sprite)
draw_sprite_ext(_sprite, _sprite, partnerX±5, partnerY-2, 1, 1, angle, c_white, 0.5)
// the partner's body
draw_sprite_ext(global.partnerSprite, global.partnerImageIndex,
                global.partnerX, global.partnerY,
                1 * global.partnerFacing, 1, global.partnerAngle, c_white, 0.5)
```

Facts this settles:

- **The puppet is drawn, not spawned.** PencilPal uses `draw_sprite_ext` in `OnDrawEnd`, not a
  game object. Simpler than the "puppet instance" the M1 placeholder comment assumed, and it
  sidesteps depth sorting for the first cut (no y-based depth to get wrong yet).
- **The ghost look is alpha `0.5`** on the partner sprite - one constant, not an art problem.
- **A held item travels as an item id** (`partnerHold`), rendered through
  `ItemGet(id, ItemData.Sprite)` - the same id-based rule as loot. Cosmetics ride along the same
  way: an id the far side turns into its own sprite. This is the "send facts, not pixels" idea
  made concrete: both installs run the same Forager, so only the ids, facing and frame travel.
- **The visual state of a puppet is four values**: `sprite`, `image_index` (animation frame),
  `facing` (±1, flips the sprite), `angle` (0 or 90 for tools). The current `POS` packet already
  carries sprite, facing and angle; **`image_index` is the missing piece** that makes a puppet
  walk instead of glide, and PencilPal's own notes flag that realtime `image_index` sync was
  "too laggy" for it and got cut - worth knowing before promising smooth animation.

Planned split: **M3.1** puppet (sprite/position/facing/angle), **M3.2** animation
(`image_index`), **M3.3** name over the head, **M3.4** held item and cosmetics. The project's
own `Events/OnDrawEnd.gml` is where the peer marker lives today (an aqua circle), so the puppet
replaces that placeholder in place.

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
