# Forager co-op mod — design (working document)

Stage: **building — the mod is playable end to end for walking around.** M0-M4 are done and run
against the live game; M5 is in progress and its longest piece (the puppet's animation) closed
2026-10-23. The world-sync design below is validated for land and structures, proposed for the rest;
API facts are sourced in `NOTES.md`. **See "What is left, in order" for the current plan.**

Code: **https://github.com/Egolz/forager-coop** (GPL-3.0). This folder is the write-up; the
mod's `.gml` source, its protocol notes and the mock peer live in that repository.

## Goal

Let 2 players share one Forager world over the internet, without port forwarding, without
the crash/desync profile of PencilPal. MVP is deliberately small; everything beyond it is
a later milestone.

## Non-goals (for the MVP)

- More than 2 players.
- Shared quest/story progression, boss fights, or the Void.
- Mods other than ours in the load order (detect and warn instead).
- Dedicated servers / host migration.

## The core decision

PencilPal let **both clients run the full simulation and broadcast diffs**. Any missed or
duplicated mutation then diverges or crashes — which matches the reported "crashes every
5 minutes / unplayable". We invert that:

**Host-authoritative, sequenced mutations.**
- The **host** owns the world. Its copy is the truth.
- A **client sends *intents*** ("I broke tile at (x,y)", "I placed a Structure at (x,y)",
  "I picked up object id N"), never results.
- The host validates each intent against its own state, applies it via the game API, and
  **broadcasts the confirmed mutation with a monotonically increasing `seq`**.
- Clients apply confirmed mutations in `seq` order and ignore duplicates/old ones.
- Positions of player puppets are the one exception: each player is authoritative for
  their own puppet position (broadcast at the system-step cadence) because movement is
  cosmetic and latency-sensitive.

## Architecture

```
        host (authoritative)                 client (thin)
   intent ─────────────────────────►   validates, applies
   ◄──────────── confirmed mutations (seq) ────────────
   ◄──────────── state snapshot on join ───────────────
   puppet positions  ◄────────►  puppet positions      (both ways, cosmetic)
```

Layers:

1. **Transport** — an abstraction with two backends:
   - `DirectTCP` (GameMaker `network_create_server` / `network_connect`, port 6511) for
     LAN and development.
   - `Steam` (Spacewar app 480 relay, per `playbook/networking-steam-spacewar.md`) for
     release. This is the piece that removes port forwarding.
   The sync layer must not care which backend is active.
2. **Session** — host creates, client joins. Handshake exchanges: protocol version, mod
   version + hash, game build id. Refuse to start on mismatch.
3. **World sync** — intent → validate → apply → broadcast(seq).
4. **Snapshot/resync** — on join, and on any detected divergence, the host streams world
   state **in chunks with per-chunk checksums**, then only sends deltas. This is the direct
   answer to PencilPal's "land/resources sync is very slow on big worlds".

## Forager-specific constraints (from the API)

- **System step.** Heavy work runs once per 20 frames. Batch network sends and state
  hashing on the same cadence; do not try to sync per frame.
- **Culling / True Instance.** Offscreen instances are deactivated. Use `TrueInstanceFind`
  (not `instance_find`) to reach them, and set `antiCull = true` on mod-owned instances
  (the puppet) so they are never culled. PencilPal's own TODO confirms it did not do this.
- **No object creation.** Mods spawn blank instances with `ModObjectSpawn` /
  `ModObjectSpawnPersistent` and attach methods with `InstanceAssignMethod`. The remote
  puppet is one such instance; its "step"/"draw" come from the mod.
- **Persistence.** Use `ModSaveDataSubmit` / `ModSaveDataFetch` and `InstanceAddVariable`
  for anything that must survive save/load (e.g. the shared vault). Keep this list short;
  the docs warn serialization has a performance cost.
- **Commands.** Register `/host`, `/join <ip>`, `/sync` via `CommandCreate` so the mod is
  usable before any menu UI exists.
- **Events to hook (minimum):** `OnStep` (pump network + apply incoming),
  `OnSystemStep` (batch outgoing + hash a slice of the world), `OnLoad` (re-attach mod
  state after a load), `OnStructureSpawn` (mirror structure creation).
- **Zones.** The world is multi-zone (`objPlayer.room`). MVP: sync only the zone the host
  is in; players in different zones see only each other's presence.

## MVP scope (what is actually shared)

| Subsystem | Authority | Notes |
|---|---|---|
| Player puppet (position, sprite, facing, held item) | each player | cosmetic, broadcast both ways |
| Terrain edits (break/place tile) | host | intent from client |
| Resource nodes (harvest, respawn) | host | identity by object id + tile coords |
| Dropped items / pickups | host | |
| Structure placement | host | |
| Item counts (gold, resources) | host | send deltas, not full inventories |
| Everything else | not synced | listed under Known issues |

## Packet format (proposal)

Extend PencilPal's `Commands` enum, but every gameplay packet carries `seq` (`u32`) and
the host also acks client intents by `intentId`. Keep binary buffers
(`buffer_create`/`buffer_write`/`buffer_tell`). Add a `/sync` **chunk checksum request** so
a client can prove or disprove divergence.

## Milestones

- **M0 — it loads.** ✅ 2026-10-19: the mod folder compiles in-game and `Trace`s a version
  string. Proves the toolchain.
- **M1 — presence over LAN.** ✅ 2026-10-20: DirectTCP host/join, the GM socket handshake,
  HELLO/WELCOME, ping/pong, presence packets and the remote marker, all verified against a
  non-GML mock peer — network, drawing and the F8 status line confirmed by the owner, plus
  clean handling of a dropped peer. Still missing for a *release*: a real sprite puppet
  rather than a drawn marker (deferred to M2, where puppet state matters).
- **M2 — host-authoritative world.** A client sends intents; the host validates, applies and
  broadcasts the confirmed mutation with a `seq`. Sliced so each piece is verifiable alone:
  - **M2.0 plumbing** — `INTENT`/`MUTATION` packets (`seq` + `intentId`), a mutation table, and
    a **self-test**: the mod creates a host server *and* opens a client socket to
    `127.0.0.1` inside the same game instance, sends a scripted intent, then asserts the
    confirmation and the resulting host state. Since Forager cannot run two instances, this is
    how both roles get exercised in one process.
  - **M2.1 structures** — place a structure as the client; the host spawns it and broadcasts
    the confirmation; both sides agree (hooks `OnStructureSpawn`). First slice: the cleanest
    event in the API. **Code written 2026-10-20** (protocol 2, packets 8–11: `INTENT_PLACE`,
    `MUTATION_PLACE`, `INTENT_REJECTED`, `STRUCTURE_TABLE`); the mock peer speaks them and its
    codec passes an offline self-test, but the two roles have **not** yet been exercised in
    the live build — the first live run is pending, so treat M2.1 as unverified.
    Decisions taken here: a structure travels by **name** (`"forge"`), not object index
    (build-independent, greppable in a log); the host validates room, structure existence and
    a 448 px distance from the asker's last reported position, then creates the structure
    itself; the client places optimistically and rolls back on `INTENT_REJECTED`.
  - **M2.2 resources and loot** — harvest nodes, dropped items, pickups
    (`OnResourceDestroy`, `OnItemDrop`, `OnItemGet`).
  - **M2.3 terrain** — tile break/place. No event exists for it, so the host must detect tile
    changes itself and turn them into mutations.
  - **M2.4 item counts** — gold and resource deltas.
- **M3 — resync.** Chunked snapshot on join + checksum-based divergence recovery.
  - **M3.1 the world snapshot — the land.** ✅ code written 2026-10-21 (protocol 6, packet
    17 `SNAP_LAND`): the host walks the world's own 16 px grid and sends every cell that is
    land, in chunks of 512, with a generation byte and an end marker; the client fills in the
    cells it is missing and, behind a flag, removes the land the host does not have. Both
    function names (`PlaceLandfill`, `RemoveLandfill`) and the whole shape come from
    PencilPal, the one mod known to run in this build — see `NOTES.md`.
    **Verified live 2026-10-21:** the game placed the cell the host sent it, and the
    self-check found no cell where the call did not take — so `PlaceLandfill(x, y)` takes
    absolute coordinates and the land sync works as designed.
  - **M3.2 the world snapshot — the buildings.** ✅ code written 2026-10-21 (packet 18
    `SNAP_STRUCT`): the host sends its structures as (position, **database id**), and the
    client adopts or builds each one. There is **no call that lists the structures in the
    world** — the database holds the types, not the instances — so the world is *watched*
    through `OnStructureSpawn`, and the list is filled before that event's guards, because
    the buildings that matter are the ones that were standing before the peer joined. Still
    missing from the snapshot: the **resources** (trees, rocks, bushes), so the two islands
    still have to have the same trees in the same places.
- **M4 — Steam relay.** Swap DirectTCP for the Spacewar relay backend; no port forwarding.
  ⏳ `tools/steamlink/` written and verified against real Steam (2026-10-22): a separate Python
  process loads `steam_api64.dll` via `ctypes` and relays, so the mod still speaks plain TCP to
  `127.0.0.1`. **The peer-to-peer path with a second account is unverified** (needs a second
  Steam account and a second machine).
- **M5 — automated QA.** Two scripted clients that perform fixed mutations and compare
  shared-state hashes; log PASS/FAIL.
  - **M5k — the puppet's animation.** The second player's body must be as smooth as the player's.
    This ran to **round 5e** and is the longest single chase in the project, because the symptom
    ("the puppet bobs too slowly") was in fact **three separate bugs stacked**:
    1. **M5j** — the walk was time-driven, so the puppet walked in place; it is now movement-driven.
    2. **M5k8** — the animation clock was `current_time` inside `OnDrawEnd`, where the handler runs
       ~505x/s and `current_time` ticks ~15 ms, so a sum of gaps came out **14.89x** the wall clock.
       The bob moved to the **step** clock, with the step length **measured**, not taken from `fps`.
    3. **M5k8c** — the rate itself was overridden by **packet noise**: a standing peer's jitter
       (~0.0007 u/ms) passed a `<= 0.0005` gate meant to mean "no reading" and beat the correct
       0.012. Fixed with a **band** (`CoopIdleUnits()`, one decision in one place) and a floor at
       the measurement.
    **Verified live by the owner 2026-10-23: "the mock bobs its head perfectly."** Full history and
    the lessons are in `NOTES.md` (rounds 1-5e).

## What is left, in order (2026-10-23)

The mod is **playable in the walk-around sense**: two players see each other, the puppet is a real
sprite with the player's own walk, dust, roll and now a correct idle bob, and land/structures/held
items/mobs sync. What follows is the honest ordered list, cheapest-and-most-visible first.

**Next up (agreed order):**

1. **The puppet's walk still stutters** — owner's report: *"the mock animates movement slowly and
   freezes"*. The walk branch in `OnDrawEnd` still advances `coopPeerFrameF` from the **same
   `current_time`** that was just proven wrong for the idle bob (14.89x). The fix is the same one
   M5k8 applied to the bob: advance it on the **step** clock. It was left alone while the bob was
   being chased only because walking was previously *reported working* — that report is now suspect.
2. **The slime's model.** `ModEnemySpawn` spawns `objModEnemy`, which has no body; it needs
   `objSlime`'s (or the named enemy's) sprite so a synced mob looks like the mob it is.
3. **The dust animation** on the puppet, and the **roll** — verified as working once, but on the same
   untrustworthy draw clock, so both should be re-checked after (1).
4. **The `Co-op: N thing(s) left the world` line** — the diff-based census still reports removals
   that are not removals (cosmetic noise in the log).

**Bigger pieces, in dependency order:**

5. **Zone awareness.** MVP syncs only the host's zone (`objPlayer.room`). Players in different zones
   currently see only presence, and the puppet is drawn even when the peer is in another room.
6. **Terrain edits (M2.3).** Tile break/place has **no event** — the host must detect tile changes
   itself. Land *snapshot* (M3.1) works; live edits do not.
7. **Resources in the join snapshot.** Trees, rocks and bushes are **not** in the snapshot, so the two
   islands must already have identical resources. `OnStructureSpawn` covers buildings; there is no
   equivalent list for natural objects, so this needs the same "watch the world" trick.
8. **The story, the Void, temples, achievements.** Deliberately out of the MVP: story/world is shared
   in principle, but bosses, the Void and progression triggers are unsynced. The owner's broader goal
   names these as *wanted*, so they are future work, not non-goals forever.
9. **Steam relay peer-to-peer verification**, then **a real two-machine test** — the one thing no
   amount of mock work can substitute for. Needs a second Steam account and a second PC.
10. **The skill/achievement/inventory question.** Each player currently runs **their own save**, so
    skills and inventory are personal by construction. Whether the *second* player can level skills
    and earn achievements in a shared world is a design question still open — see `NOTES.md`.

**Known environment limits (not bugs):**

- Forager refuses to run twice on one PC, so full two-client tests need a second machine or VM. Every
  result so far is *one real client + the mock peer*, and that is stated honestly everywhere.
- The owner must press **COMPILE** in the in-game mod menu; the build cannot be reloaded from outside.

## Testing

Per `playbook/automated-qa.md`: two real clients, driven by an in-mod bot (not fake input),
asserting shared state.

**Answered 2026-10-20: Forager cannot run two instances on one PC** (a second launch while
it runs is refused). So a full two-client test needs a second Windows session/VM, or a
second machine. Until then: the transport is covered by the non-GML mock peer, and the
in-mod bot can at least drive one side and assert the packets/state it gets.

## Open risks

- ~~Two instances on one PC may be impossible~~ — **confirmed impossible** 2026-10-20; see
  Testing.
- Whether a Forager mod can load the Galaxy/Steamworks GML library for the relay backend
  (`unknown`).
- The live build's API may differ from the 1.0.1 the docs describe.
