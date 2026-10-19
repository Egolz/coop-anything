# Forager (2019)

| | |
|---|---|
| Engine | **GameMaker Studio 2** (data file `data.win`) |
| Versions tested | **Steam build 6160665** — modding toolchain **and** the M1 DirectTCP network layer run in-game; no co-op world sync yet |
| Projects | PencilPal / PenPal (dead, see below); no maintained project found; **this is an open gap being worked** |
| QA level | none |
| Last updated | 2026-10-19 |

## Where things stand

There is **no working way to play Forager in co-op today**. The official multiplayer was
built, beta-tested and then cancelled in 2021 for instability. The community mod
(PenPal → PencilPal) got two players into one world but was removed from Steam Workshop
and is reported to crash constantly. The realistic options are streaming the host's game
(Steam Remote Play Together / Parsec) or building a new mod — which is what this game
folder is for.

## Existing work

| Project | Author | State | What it really syncs | Sources |
|---|---|---|---|---|
| **official multiplayer** | HopFrog | **cancelled 2021** (too buggy; refunds offered) | players, world — never shipped | https://www.rockpapershotgun.com/forager-cancels-multiplayer , https://techraptor.net/gaming/news/forager-developers-respond-to-multiplayer-cancellation |
| **PenPal** | topherlicious | dead | only rendered the friend's sprite | https://yal.cc/forager-retrospect/ |
| **PencilPal (BETA)** | "Two" (SteamID64 `76561198160202307`) | **removed from Workshop**, last update 2023-04-08, "incompatible with Forager" | host/join by typed IP on TCP 6511; syncs land, resources, dropped items, object health, chests, industrial structures, NPC quests, slimes, drones, a shared vault, a player puppet, gold, chat, emojis, a PvP mode | https://steamcommunity.com/sharedfiles/filedetails/?id=2843162754 , https://playco-opgame.com/multiplayer-mod/forager-multiplayer-mod/ |
| **Foragipelago** | community | alive (Archipelago integration) | not co-op, but proves Forager mods can talk to an **external server over sockets** | Workshop item 3760668082 (page did not render for us) |
| **GMTogether** | YellowAfterlife | alive, but **does not apply** | injects online play into GameMaker games **that already have local multiplayer**; Forager has none | https://yellowafterlife.itch.io/gmtogether , https://yal.cc/docs/gm/gmtogether/ |

The authoritative explanation of why Forager is hard comes from **YellowAfterlife**, who
wrote Forager's modding API and its (cancelled) networking. His retrospective:
https://yal.cc/forager-retrospect/ — key line: *"if the work started from local
multiplayer, the game could still be played over internet using Parsec, Remote Play
Together, or a tool I made. But it was not, so it cannot be, and at most someone might
make a slightly more advanced version of topherlicious' PenPal mod."*

## How the game works (for co-op)

**Engine / scripting.** Forager is GameMaker Studio 2. It ships an **official modding
API**: a mod is a folder of `.gml` files with a `fmConfig.json` manifest, compiled by the
game itself (`rmModCompile`, and `/recompile` from the main menu). Documented API mirror:
**https://topheranselmo.com/forager-modding-docs/** (by *lazyeye*, built by *topherlicious*;
the original was `http://138.197.196.157/docs/`). Official modders' tooling: the *Forager
Modding Tool* and a *GMEdit dialect pack* linked from that docs site.

**No netcode.** Forager is single-player only. It has **no local multiplayer**, which is
why generic GameMaker online-injection tools cannot be used.

**What the mod API gives you** (from the docs and from PencilPal's source):
- Spawning/logic: `ModObjectSpawn`, `ModObjectSpawnPersistent`, `ModNPCSpawn`,
  `ModEnemySpawn`, `ModStructureCreate`, `ModZoneCreate`, `InstanceAssignMethod`,
  `InstanceAddVariable` (marks vars to serialize), `ScriptWrap`, `TeleportObject`,
  `CollisionCheck`, etc. You cannot create new objects — you spawn blank instances and
  attach methods.
- Events to hook: `OnSystemStep`, `OnLoad`, `OnStep`, `OnEvent`, OnStructureSpawn`,
  `OnChatMessage`, `OnLevelUp`, `OnNetworkAsync`, `OnDrawEnd`, `OnDrawGUIEnd` (exact set
  is documented under *Native/Events*).
- Player/world read: `objPlayer` (x/y, `room`, sprite, holding), `Item.*` / `ItemData.*`
  enums, `ItemCount` / `RemoveItem` / `ItemEdit`, `Structure.*` (e.g. `Structure.Forge`),
  `Skill.*`, `HasSkill`, `WaterCollision`, `LighthouseCalculate`.
- Persistence: `ModSaveDataSubmit` / `ModSaveDataFetch` (used by PencilPal to sync chests);
  `InstanceAddVariable` for serialization.
- Chat/console: `Trace(...)`, `CommandCreate(...)` for custom `/commands`.
- In-game dialogs: `get_string`, `get_integer` (standard GML) — PencilPal asks for the
  peer's IP this way.
- **Networking:** the API does not add netcode, but mods can call GameMaker's built-in
  socket functions. PencilPal uses `network_create_server` / `network_create_socket` /
  `network_connect` + `network_send_packet` over raw TCP on port 6511, with binary
  `buffer_*` packets and an `OnNetworkAsync` handler.

**Performance shape that matters for sync.** Forager runs a **"system step"** — heavy work
happens once every 20 frames, spread across frames (`OnSystemStep`), not every frame.
Offscreen instances are **culled** (deactivated); a custom **True Instance** system lets
game code still reach them (`TrueInstanceFind` and friends). Any co-op design must respect
culling and True Instance, and should not assume per-frame access to everything.

**Save system.** Forager has a save system; mods can attach save data via
`ModSaveDataSubmit`/`ModSaveDataFetch` and `InstanceAddVariable`. *Exact save file layout
is `unknown` — to be mapped in the Understand phase.*

## What we changed

**A working M1 network layer, verified in the live build.** Contributions are the triage,
the PencilPal source analysis, the API map above, and two run milestones:

- **M0** — a scaffold mod (`fmConfig.json` + `Main.gml`) compiles and loads in the live build
  6160665 using only the game's own loader: no external tool, no `data.win` editing.
- **M1** — DirectTCP host/join (F5/F6), a version-checked HELLO/WELCOME handshake, ping/pong
  keepalive and player-presence packets. Verified against a non-GML peer, which also pinned
  down **GameMaker's own TCP handshake and packet framing** (recorded in `NOTES.md`) — the
  piece any non-GML peer or relay must implement.

No world sync yet: that is M2. The mod's code lives in its own repository (linked from here
once it exists); this folder holds the write-up only.

## QA

QA level: **none** for co-op — nothing playable is synced yet. The one thing actually
asserted is the M1 **transport**: a scripted mock peer opened a DirectTCP session against
the running game and completed the GameMaker socket handshake, the mod's HELLO/WELCOME
exchange and bidirectional ping/pong. That is a transport test, not a co-op test. Real co-op
QA still needs two Forager clients driven by an in-mod bot (see `playbook/automated-qa.md`)
— and whether Forager can run two instances on one PC is still `unknown`.

## How to play

Not possible yet. Interim workaround for playing *together* today, without a mod:
**Steam Remote Play Together** (or Parsec) — the host runs the game, friends join the
stream and use controllers/keyboard. This does not make it a true co-op mod, but it needs
no code and no port forwarding.

## Known issues (inherited from PencilPal, to design against)

- Constant crashes and desync; comments into late 2025 call it "unplayable".
- Requires the host's **TCP port 6511 forwarded** (no relay).
- Host is not synced with industrial structures; forge weapon upgrades never sync.
- Land and resource syncing is very slow/intensive on large worlds (the mod's own TODO).
- "Changing menu crashes" / "changing rooms deletes inventory" — reported but not
  reproduced by the author.
- Because PencilPal ships both clients running the full simulation and exchanging diffs,
  a missed or duplicated mutation desyncs or crashes. A new design should be
  **host-authoritative with sequenced mutations** (client sends intents, host resolves).

## Next steps

1. **Identity** — done: Steam build **6160665**, `Forager.exe` and `data.win` SHA-256 recorded
   in `NOTES.md`. Still `unknown`: whether two Forager instances can run on one PC (needed for
   automated two-client QA).
2. **Understand** — read the full API docs (Events, Saving, Spawn/Structure/Zone
   databases, Assets, Macros), dump the API surface of the live build, and map the save
   file.
3. **Choose the networking model** — start from PencilPal's protocol but make it
   host-authoritative + sequenced, keep the GML sync layer, and move transport to
   **Steam relay via Spacewar (app 480)** per `playbook/networking-steam-spacewar.md`
   so no one forwards a port.
4. **Scope an MVP** — 2 players, one shared world, sync only: player puppet, terrain
   edits, resource pickups, dropped items, structure placement. Everything else later.
5. **Scaffold the mod repo** — `fmConfig.json` + `Main.gml`, GPL-3.0 (the licence used by
   the community example `Zamiell/FastCrafting`); other examples: `CainDev/Forager-Mods`,
   `AlejandroAkbal/Forager-Cheater-Mod`. Design in [`DESIGN.md`](DESIGN.md).
6. **Automate QA** — a scripted two-client test comparing agreed shared state.
