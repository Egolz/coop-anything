# Forager co-op mod — design (working document)

Stage: **design only.** No code has been built or run. Everything here is a proposal to
be validated against the live game; API facts are sourced in `NOTES.md`.

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

- **M0 — it loads.** Mod folder compiles in-game and `Trace`s a version string. Needs a
  restored legit copy. Proves the toolchain.
- **M1 — presence over LAN.** Two clients, DirectTCP, puppets visible and moving. Clean
  connect/disconnect (no crash if the peer drops).
- **M2 — host-authoritative world.** Terrain edits, resource pickups, dropped items,
  structure placement via intents, with `seq` ordering.
- **M3 — resync.** Chunked snapshot on join + checksum-based divergence recovery.
- **M4 — Steam relay.** Swap DirectTCP for the Spacewar relay backend; no port forwarding.
- **M5 — automated QA.** Two scripted clients that perform fixed mutations and compare
  shared-state hashes; log PASS/FAIL.

## Testing

Per `playbook/automated-qa.md`: two real clients, driven by an in-mod bot (not fake input),
asserting shared state. First we must answer: **can Forager run two instances on one PC?**
If not, QA needs two Windows sessions/VMs.

## Open risks

- Two instances on one PC may be impossible (`unknown`).
- Whether a Forager mod can load the Galaxy/Steamworks GML library for the relay backend
  (`unknown`).
- The live build's API may differ from the 1.0.1 the docs describe.
