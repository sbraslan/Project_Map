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


## Character / party membership lifecycle closure

`CHARACTER::SetDungeon(newDungeon)` is the authoritative character-side bridge.

On leaving an existing dungeon:
- PC with party -> `oldDungeon->DecPartyMember(GetParty(), this)`;
- PC without party -> `oldDungeon->DecMember(this)`;
- monster/stone -> `oldDungeon->DecMonster()`.

On entering:
- PC with party -> `newDungeon->IncPartyMember(GetParty(), this)`;
- PC without party -> `newDungeon->IncMember(this)`;
- monster/stone -> `newDungeon->IncMonster()`.

Character destruction calls `SetDungeon(nullptr)` if needed.

Party removal is deliberately ordered so dungeon bookkeeping sees the old party pointer:
`CHARACTER::SetParty(nullptr)` first calls `SetDungeon(nullptr)` while `m_pkParty` is still set, then clears the party pointer.

`CDungeon::IncPartyMember` increments `m_map_pkParty[pParty]` and also inserts the character into the general dungeon character set through `IncMember`.

`DecPartyMember` decrements that count, and when it reaches zero calls `QuitParty(pParty)`, which clears the party's generic dungeon pointer and erases the raw party key. It then calls `DecMember(ch)`.

Therefore the previously suspected normal party-destruction dangling `m_map_pkParty` key closes on the mapped online-member lifecycle: leader/member `SetParty(nullptr)` transitions decrement the dungeon party count before the `CParty` object is deleted.

## Warp/login dungeon binding
Party/single-character dungeon warp helpers do not directly call `SetDungeon(this)` before `WarpSet`.

On game entry after warp, `input_login.cpp` resolves:
`CDungeonManager::FindByMapIndex(ch->GetMapIndex())`
and calls:
`ch->SetDungeon(resolvedDungeon)`.

That establishes the character/dungeon membership and party count on the destination game process.

## Candidate closures
### JoinParty map-existence ordering
`JoinParty` sets party/dungeon pointers before checking `SECTREE_MANAGER::GetMap(m_lMapIndex)`.

Normal manager lifecycle does not expose a live `CDungeon` without its private map:
- `Create` constructs the dungeon only after `CreatePrivateMap` succeeds;
- `Destroy` destroys the private map immediately before deleting the dungeon object.

No normal source path was found that removes the private map while retaining the dungeon object. This remains defensive-ordering debt, not a promoted bug.

### Exit/jump event lookup null ordering
`dungeon_jump_to_event` and `dungeon_exit_all_event` assign through the manager lookup result before checking it for null.

However `CDungeon::~CDungeon` cancels both `jump_to_event_` and `exit_all_event_`, and manager destruction deletes the object through that destructor. No normal lifecycle path has yet been found where one of these events survives after its dungeon is removed from the manager.

The null ordering is incorrect but remains unpromoted until a live stale-event path is established.

## Quest create/join lifecycle

The active build enables both:
- `D_JOIN_AS_JUMP_PARTY`;
- `ENABLE_D_NJGUILD`.

### BUG-DUNGEON-001 — quest entry APIs can orphan a newly-created private dungeon after post-create eligibility failure

`dungeon_join` performs:
1. validate Lua map argument;
2. `CDungeonManager::Create(lMapIndex)`;
3. obtain current character;
4. under active `D_JOIN_AS_JUMP_PARTY`, require that character to have a party;
5. if no party, log `cannot go to dungeon alone.` and return.

Thus a solo player calling the registered `d.join(mapIndex)` API causes the private map/dungeon to be created **before** the function rejects the player.

The new dungeon has:
- empty `m_set_pkCharacter`;
- `deadEvent == nullptr` from initialization.

Automatic dead destruction is only scheduled inside `DecMember` when a previously tracked member is removed and the set becomes empty. A dungeon that never receives a member never reaches that scheduling point.

Therefore the rejected call leaves an orphan private dungeon/map registered in `CDungeonManager` with no automatic destruction event.

The same create-before-eligibility pattern exists in active `dungeon_new_jump_guild`:
- creates private dungeon first;
- then resolves current character;
- then checks `ch->GetGuild()`;
- a non-guild caller returns after allocation.

This creates BUG-DUNGEON-001.

Current source dungeon quests primarily use `d.new_jump_party` / `d.new_jump_all`, whose mapped common call paths do not demonstrate this failure. Impact of the exposed `d.join` / `d.new_jump_guild` APIs therefore depends on quest usage, but the resource leak is deterministic for the rejected invocation itself.

## Exact next audit
1. Audit `JumpParty` one-party ownership pointer across nested/repeated dungeon creation.
2. Close dead/exit/jump event lifetime and manager-ID reuse more deeply.
3. Audit participant registration and member sets under cross-core/private-map warp.
4. Audit quest item-group removal and entry-item lifecycle.
5. Continue spawn/unique/regen pointer lifecycle and promote only verified bugs.


## Participant registry closure
Under active `ENABLE_DUNGEON_RENEWAL`, participant state is:
`std::map<uint32_t, std::string> m_Participants`.

It stores PID and copied name only, not `LPCHARACTER`. Register/check/clear operations therefore do not retain character pointers across logout/core transitions. No dangling-character defect was found in this registry.

## Unique mob pointer lifecycle closure
`m_map_UniqueMob` does store raw `LPCHARACTER`, but the mapped destruction paths clean it:
- normal dungeon mob death -> `CHARACTER::Dead -> GetDungeon()->DeadCharacter(this)`;
- generic `CHARACTER_MANAGER::DestroyCharacter` also calls `dungeon->DeadCharacter(ch)` for dungeon monsters/stones before physical destruction;
- `PurgeUnique` and `KillUnique` erase the unique-map entry before destroying/killing the character.

`DeadCharacter` searches the unique map and erases the matching pointer. No additional normal unique-mob dangling pointer bug was found.

## Live item-group quest path / cross-system reachability
Devil Catacomb has an active timer flow:
- `d.set_item_group("reapers_credit", ...)`
- `d.exit_all_by_item_group("reapers_credit")`
- `d.delete_item_in_item_group_from_all("reapers_credit")`.

The item-group deletion path uses the same `CountSpecifyItem/RemoveSpecifyItem` primitives audited in Party Match.

Relevant item vnums are 30319, 30320 and 76002. Their runtime item-proto anti-give/exchange flags are not present in the mapped repositories, so exchangeability of these specific items cannot be proven statically. The Party Match exchange UAF is therefore not duplicated as a Dungeon bug without that missing proto fact.

The same Devil Catacomb `exit_all_by_item_group` path has a different confirmed cross-system consequence:
- for a party member without the required item, it may call `pParty->Quit(ch->GetPlayerID())` when party size is greater than 2;
- if that member is the party leader, it reaches the already verified BUG-PARTY-001 self-delete/use-after-free path.

This adds a live dungeon-quest reachability path to BUG-PARTY-001 but is not assigned a duplicate Dungeon bug ID.

## Dungeon Lua getter audit
The following Lua getters contain a suspicious validation guard requiring two numeric Lua arguments even though the implementation does not use those arguments:
- `d.get_kill_stone_count`
- `d.get_kill_mob_count`
- `d.is_use_potion`
- `d.revived`.

An arg-less call therefore returns the fallback value rather than current dungeon state.

However all 98 non-generated source `.quest` files under `share/locale/europe/quest` were statically checked and no current call to these four APIs was found. They remain dormant API defects, not promoted current-gameplay bugs.

## Exact next audit
1. Audit regen list/event lifetime and ClearRegen ordering.
2. Audit spawn/purge/kill bulk operations under pending-destroy iteration.
3. Audit dungeon unique alias edge cases and SetUnique multi-key behavior.
4. Close nested JumpParty ownership as dormant vs reachable.
5. Continue toward Dungeon Core STATIC COMPLETE.


## Regen lifetime audit — CLOSED
**Tarih:** 2026-09-27

Dungeon regen lifetime was traced end-to-end through `regen.cpp`, `dungeon.cpp` and `CHARACTER::Destroy`.

For persistent dungeon regen:
- `regen_do` allocates a `REGEN`, stores the dungeon ID in `dungeon_regen_event_info`, creates the event, and registers the pointer through `CDungeon::AddRegen`;
- `ClearRegen` cancels each `regen->event`, deletes the `REGEN`, then clears `m_regen`;
- `dungeon_regen_event` resolves the dungeon by ID before using its regen payload;
- spawned characters keep both the raw regen pointer and the copied `regen_id_`;
- `CHARACTER::Destroy` does not dereference the stored regen until `CDungeon::IsValidRegen(pointer,id)` succeeds;
- `IsValidRegen` first checks that the pointer still exists in the current dungeon regen vector and only then reads its ID.

Therefore a character that survives `ClearRegen` may still retain the old pointer value, but the current destruction path rejects it without dereferencing the freed `REGEN`. No additional verified bug was promoted from this path.

## Bulk purge / kill iteration audit — CLOSED
`CDungeon::KillAll`, `Purge` and `KillMonsters` iterate through `SECTREE_MAP::for_each`.

That helper first collects entities into an `FCollectEntity` snapshot and only then invokes the destructive callback. Removing characters/items from their sectree during the callback therefore does not invalidate the map traversal itself.

`CHARACTER_MANAGER::DestroyCharacter` also prevents duplicate destruction by checking the VID map, and supports deferred destruction when the pending-destroy mode is active.

No deterministic Dungeon Core iterator invalidation was established for the mapped bulk purge/kill paths.

A lower-level `SECTREE::for_each_entity` stale-relationship branch erases from its entity set in-place with questionable iterator handling, but no normal path producing that stale relationship has been mapped. It remains unpromoted defensive debt.

## Unique alias audit

### BUG-DUNGEON-002 — SpawnMoveUnique can spawn up to 100 mobs for one requested unique key
`CDungeon::SpawnMoveUnique` has a 100-attempt loop.

On a successful spawn it:
- inserts `key -> ch` into `m_map_UniqueMob`;
- marks the mob with the dungeon-unique affect;
- binds it to the dungeon;
- sends it toward the target area;
- but does **not** break or return.

The loop therefore continues after success. If spawning keeps succeeding, a single `d.spawn_move_unique(...)` call can create up to 100 mobs.

Because `m_map_UniqueMob` is a `std::map` and insertion uses `insert`, only the first successful pointer is registered for that key. Later mobs remain alive in the dungeon but are not addressable through the requested unique key.

The Lua API is registered as `d.spawn_move_unique`, so the defect exists at the public dungeon-script surface.

### BUG-DUNGEON-003 — multi-key SetUnique aliases can leave dangling raw pointers
`CDungeon::SetUnique(key, vid)` permits the same character VID to be inserted under multiple different string keys. No uniqueness-by-pointer invariant is enforced.

`CDungeon::DeadCharacter(ch)`, however, scans `m_map_UniqueMob`, erases only the **first** entry whose pointer equals `ch`, then breaks.

Consequences depend on destruction path:
- direct destruction through generic `CDungeon::Purge` reaches `CHARACTER_MANAGER::DestroyCharacter` once; with two aliases, only one is removed before the character object is deleted and another alias remains stale;
- normal mob death calls `DeadCharacter` once from `CHARACTER::Dead`, and later destruction calls it again; three or more aliases still leave at least one stale entry.

Consumers including `GetUniqueVid`, `IsUniqueDead`, `GetUniqueHpPerc`, `UniqueSetMaxHP`, `UniqueSetHP` and `UniqueSetDefGrade` dereference the stored raw pointer.

Therefore registered Lua `d.set_unique` can construct a dangling-pointer/UAF surface by assigning one mob to multiple unique keys.

## Nested JumpParty ownership — still unpromoted
`CDungeon::JumpParty` enforces ownership only while `pParty->GetDungeon_for_Only_party() == nullptr`.

If that pointer is already non-null, the function does not check whether it equals the destination dungeon and proceeds to warp matching party members. This can theoretically place a party into Dungeon B while its exclusive pointer still names Dungeon A and while Dungeon B never sets `m_pParty`.

The semantic invariant is weak, but no current source quest nested/re-entry call chain has yet been established. It remains a candidate until live source-script reachability is found.

## Exact next audit
1. Scan current source quests for live `d.spawn_move_unique` / multi-key `d.set_unique` usage and nested `d.new_jump_party` transitions.
2. Close duplicate-key behavior for `SpawnUnique` / `SetUnique` and classify current-script reachability.
3. Revisit dungeon manager ID wrap/stale-event identity safety.
4. Close remaining eliminate-event null-ordering candidate.
5. Decide Dungeon Core STATIC COMPLETE.


## Dungeon ID / event identity audit — CLOSED
`CDungeon::IdType` is `uint32_t`; `CDungeonManager::Create` increments `next_id_` and, after wrap, skips any ID that is still present in the live dungeon map.

Mapped dungeon-owned event lifetimes:
- `deadEvent`
- `exit_all_event_`
- `jump_to_event_`
- per-REGEN dungeon events.

`CDungeon::~CDungeon` cancels the three object events and `ClearRegen` cancels all regen events. `CDungeonManager::Destroy` also cancels quest server timers keyed by the private map before deleting the object.

`event_cancel` marks queued elements cancelled; if cancellation occurs while an event is processing, it sets `is_force_to_end` and cancels any queued element.

The dead event nulls its own dungeon event pointer before manager destruction. For exit/jump events, the callback's null-check ordering is syntactically wrong because it writes through `pDungeon` before testing it, but the mapped object lifecycle cancels those events before the dungeon can disappear from the manager. No normal stale event -> reused-ID path was established.

Result: manager ID wrap/event identity is closed without a new verified bug. The incorrect null ordering remains defensive code debt only.

## Duplicate unique-key audit

### BUG-DUNGEON-004 — duplicate unique key silently creates/mutates an unregistered entity
`SpawnUnique`, `SpawnMoveUnique` and `SetUnique` all register with:
`m_map_UniqueMob.insert(make_pair(key, ch))`
and do not test the insertion result.

For `SpawnUnique`:
1. call once with key K -> mob A is spawned and registered;
2. call again with the same key K -> mob B is spawned;
3. map insertion fails because K already exists;
4. code still binds B to the dungeon and applies `AFFECT_DUNGEON_UNIQUE`;
5. registry K still points only to A.

Thus B is a live "unique"-marked dungeon mob that cannot be addressed through K. Killing/purging K operates on A only.

For `SetUnique`, assigning an already-used key to a different VID likewise leaves the map pointing to the old character while still applying the unique affect to the new character.

This is separate from BUG-DUNGEON-002:
- BUG-DUNGEON-002 is one `SpawnMoveUnique` call multiplying spawns because success does not break the loop;
- BUG-DUNGEON-004 is key-collision handling across registration attempts.

The Lua surfaces `d.spawn_unique`, `d.spawn_move_unique` and `d.set_unique` are all registered.

## Current Dungeon Core verified set
- BUG-DUNGEON-001 — rejected entry orphan private dungeon.
- BUG-DUNGEON-002 — SpawnMoveUnique success does not stop the 100-attempt loop.
- BUG-DUNGEON-003 — multi-key alias can leave dangling raw unique pointer.
- BUG-DUNGEON-004 — duplicate unique key creates/mutates an entity not represented by the registry.

## Exact next audit
1. Establish current quest-script reachability for unique APIs and nested `d.new_jump_party` without bulk-reading the whole quest tree.
2. Audit remaining dungeon count/eliminate invariants around `m_iMonsterCount` and direct purge/death paths.
3. Audit private-map destroy ordering against character `SetDungeon(nullptr)` on map teardown.
4. Decide whether any remaining candidate is promotable.
5. Move Dungeon Core toward STATIC COMPLETE.
