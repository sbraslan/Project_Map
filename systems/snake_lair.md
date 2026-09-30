# Snake Lair / Queen Nethis — Static System Map

**Status:** STATIC MAPPING OPEN / 4 VERIFIED BUGS
**Mode:** detection / mapping only
**Execution:** LOCKED / NOT RUN

## Canonical scope
This node owns the enabled Queen Nethis / Snake Lair system:
- `ENABLE_QUEEN_NETHIS`;
- `SnakeLair::CSnk` and `CSnkMap`;
- portal entry, private-map ownership and party binding;
- floor progression, pillar/blacksmith/statue/item mechanics;
- Sungma dungeon attributes;
- kill, party-leave, input and item interaction hooks.

White Dragon / Alastor and DawnMist / Temple of Ochao are separate canonical families.

## Live deployment proof
- The feature macro is enabled under `ENABLE_YOHARA_SYSTEM`.
- Character NPC-click handling directly calls `SnakeLair::CSnk::Start()` for the Snake portal.
- Character combat, input and party code call SnakeLair lifecycle methods.
- SnakeLair Lua functions are registered.
- Relevant server and map/data assets are tracked.

## Audit cursor
1. Map portal access, party requirements, private-map creation and registration.
2. Audit floor/state progression, timer/event ownership and item-based transitions.
3. Audit disconnect, party-leave, kick/exit and destruction ordering.
4. Audit Sungma calculations, boss/debuff logic and client/server parity.
5. Close deployment/data parity and promote only source-proven defects.

## Cursor 1 checkpoint — entry / registration ownership
- `SnakeLair.Access()` is the private-map creation API. It requires a party and party leader, creates a private copy of `MAP_SNAKE_TEMPLE_02`, registers `CSnkMap*` by private map index, and warps only same-map party members.
- The private instance constructor spawns portal VNUM 4022; clicking that portal is wired directly in `CHARACTER::OnClick` to `CSnk::Start()`, which binds the current party and starts floor 2.
- The tracked Game map spawns NPC 20807 on Snake Temple 01, but the pinned quest source/object tree does not expose a tracked `SnakeLair.Access()` caller or a 20807 quest-object binding. Full outer-entry requirements are therefore a deployment/source-completeness gap and are not invented from C++.
- `Access()` has a recall-position failure branch after private-map allocation/registration that returns success without immediate cleanup. The pinned Snake Temple 02 has a valid Town recall entry, so this branch is not promoted as an active snapshot defect.
- Same-map party collection is consistent with the actual warp set; off-map party members are neither collected nor warped.

### Closed constructor candidate
The earlier constructor-pointer suspicion was rejected after tracing the event type: `LPEVENT` is `boost::intrusive_ptr<EVENT>`, not a raw pointer. Its default constructor runs before the `CSnkMap` constructor body and initializes the handle to null. The initial `event_cancel` guards therefore do not read indeterminate raw pointers and are **not** a defect.

## Cursor 2 findings — floor/item progression

### BUG-SNK-001 — wrong-order pillar use destroys the valid key
`CSnkMap::OnKillPilar` removes the 70422 pillar key before validating the required pillar order. Pillars 2-6 then reject an out-of-order attempt with an early return, but the key has already been destroyed.

### BUG-SNK-002 — statue interaction destroys items before target/element validation
The outer statue-item filter is impossible:
`itemVnum < SNAKE_STATUE1 && itemVnum > SNAKE_STATUE4`.
It can never reject any item and also compares item VNUMs to statue NPC VNUMs. The inner handler then removes the item before checking whether the target statue is already locked or whether the item element matches that statue. Arbitrary/wrong items can therefore be lost without progress.

### BUG-SNK-003 — Ice Siren counter completes the floor on the first kill
Step-4 substep 11 stores the number of spawned Ice Sirens in `KillCountMonsters`. On a Siren kill, the code saves that value as `remainSirens`, increments the counter, and tests `newCount >= remainSirens`. For any spawned count 1-4, that condition is true on the first kill, so the dungeon advances to floor 5 immediately instead of requiring all spawned Sirens.

### Event ownership checkpoint
- `r_snakespawn_event`, `r_snakelimit_event` and `r_snakeskill_event` hold `CSnkMap*` in their event-info objects.
- Completed `LPEVENT` objects are safely retained by their intrusive owner handle with `q_el == nullptr`; later `event_cancel` simply clears the handle. No dangling-event-handle defect was promoted from this pattern.
- Stage transitions cancel/replace the spawn event before installing the next progression event.

### Next cursor
Disconnect, party-leave, kick/exit and destruction ordering.


## Cursor 3 finding — party destruction orphans the private Snake instance
`CParty::Destroy()` contains explicit Snake handling for connected members. For a member currently on a Snake map it calls:
1. `CSnk::LeaveParty(mapIndex)`;
2. `CSnk::Leave(character)`.

`LeaveParty` only erases the map index from `m_dwRegGroups`. It does **not** call `CSnkMap::EndDungeonWarp`, destroy the private sectree, cancel Snake events or delete the `CSnkMap` object. The subsequent `Leave` only warps that connected character out.

The private map, monsters and its spawn/skill/end events therefore continue to exist without a manager registration until the original one-hour `r_snakelimit_event` finally calls `EndDungeonWarp`.

Promoted as `BUG-SNK-004`.

### Other lifecycle checks
- Normal character disconnect does not destroy the party; it marks the party member offline and unlinks the character, allowing normal party relink on reconnect.
- Voluntary party leave/kick packets are blocked while the requester is on a Snake map.
- `EndDungeonWarp` cancels owned events in `Destroy()`, destroys the private sectree, removes the registration and deletes the `CSnkMap`; no additional teardown UAF was proven.
- Event callbacks that terminate while retained in an `LPEVENT` owner are safe because `LPEVENT` is intrusive and completed events carry `q_el == nullptr`.
- Several global Snake wrappers use `m_dwRegGroups.find(map)->second` without an end check. In the mapped normal orphan branch, party destruction also clears/warps connected party members, and the wrappers require a live party before the unsafe lookup. A normal post-removal crash path was therefore not promoted from that pattern.

### Next cursor
Sungma calculations, Queen Nethis boss/debuff logic and client/server parity.
