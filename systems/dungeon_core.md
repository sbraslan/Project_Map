# Dungeon Core

**Status:** PARTIAL — ACTIVE
**Phase:** Detection / Mapping Only
**Date:** 2026-09-26

> This is the generic private-dungeon engine, not the already-completed Dungeon Info UI subsystem and not feature-specific dungeons such as Zodiac/Meley/Snake.

## Source roots
- `game/src/dungeon.h`
- `game/src/dungeon.cpp`
- `game/src/questlua_dungeon.cpp`
- `game/src/char.cpp::SetDungeon`
- `game/src/party.cpp` dungeon link helpers
- `game/src/sectree_manager.*` private-map lifecycle
- quest/server timer integration in `questmanager.*`

## Core manager model
`CDungeonManager` maintains:
- dungeon ID -> `CDungeon*`
- private map index -> `CDungeon*`
- monotonic/wrapping `next_id_`.

`Create(originalMapIndex)`:
1. asks `SECTREE_MANAGER` to create a private copy;
2. allocates a unique dungeon ID;
3. creates `CDungeon(id, originalMap, privateMap)`;
4. indexes it by both ID and private map.

`Destroy(id)`:
1. removes both manager mappings;
2. cancels quest server timers keyed by private map index;
3. destroys the private sectree map;
4. deletes the `CDungeon` object.

## Dungeon object state
Mapped state includes:
- original map + private map;
- raw character membership set;
- party -> member-count map;
- unique mobs/areas/flags/item groups;
- regen list;
- monster count;
- eliminate exit/warp state;
- dead/exit/jump events;
- optional one-party/one-guild ownership pointers.

## Character membership lifecycle
`IncMember(ch)` inserts the character and cancels pending dungeon-dead destruction.

`DecMember(ch)` removes the character; when the character set becomes empty it schedules `dungeon_dead_event`, which later asks `CDungeonManager` to destroy the dungeon.

`CHARACTER::SetDungeon` is part of the authoritative character <-> dungeon association and must be cross-audited against these counters.

## Party entry
`CDungeon::JoinParty(pParty)`:
- sets the party dungeon pointer;
- inserts the party into dungeon party map;
- verifies private map exists;
- warps each online member to the private map spawn.

`QuitParty` clears the party's dungeon pointer and erases it from the dungeon party map.

Generic `Join(ch)` saves exit location and warps one character into the private map.

## Movement / completion surface
Mapped operations include:
- `JumpAll`
- `WarpAll`
- `JumpParty`
- `ExitAll`
- `ExitAllToStartPosition`
- eliminate-time exit/warp scheduling
- target-dungeon jumping via private map index
- regen spawn/clear
- unique mob management.

## Quest Lua surface
`questlua_dungeon.cpp` exposes a large script API for:
- flags;
- notices/syschat/big notice;
- regen;
- spawn/purge/kill;
- create/join/jump/warp;
- party/guild entry;
- exit and eliminate actions;
- unique mobs;
- item groups;
- member queries and quest flags.

This layer is an authority/lifetime boundary because quest code drives raw `CDungeon*`, party and map lifecycle operations.

## Initial candidates under audit
- eliminate-event callbacks assign through `pDungeon->...event_ = nullptr` before checking `pDungeon` for null; reachability after valid event cancellation must be closed before promotion;
- `JoinParty` mutates party/dungeon ownership before checking that the private sectree map still exists;
- party lifetime vs dungeon raw party-map keys requires closure, especially around leader/disband paths already found in core Party;
- several source comments explicitly mark historical dungeon/party crash-sensitive areas and must be validated rather than treated as proof.

## Exact next audit
1. Map `CHARACTER::SetDungeon` <-> IncMember/DecMember and private-map transfer lifecycle.
2. Map party JoinParty/QuitParty/member-count ownership across party destruction.
3. Close dungeon dead/exit/jump event lifetimes and null ordering.
4. Map quest create/join/new_jump/new_jump_party call chains.
5. Start verified Dungeon Core bug registry.
