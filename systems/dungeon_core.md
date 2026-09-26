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
