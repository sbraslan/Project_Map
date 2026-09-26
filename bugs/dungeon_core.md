# Dungeon Core — Bug Registry

**Phase:** Detection / Mapping Only

Verified Dungeon Core bugs are listed below.

## Candidates under closure
- eliminate event callbacks dereference manager lookup result before null check;
- JoinParty sets party/dungeon state before private-map existence validation;
- raw party pointers in dungeon party map across party destruction;
- quest-driven private-map lifecycle ordering.

Candidates remain unnumbered until normal reachability/impact is statically closed.


### BUG-DUNGEON-001 — rejected quest entry can leak an empty private dungeon indefinitely
- Statik durum: **doğrulandı**
- Sınıf: private-map lifecycle / resource leak
- Active build: `D_JOIN_AS_JUMP_PARTY`, `ENABLE_D_NJGUILD`

Primary deterministic path:
`d.join(mapIndex)`
-> `CDungeonManager::Create(mapIndex)`
-> private map + `CDungeon` registered
-> resolve current character
-> active `D_JOIN_AS_JUMP_PARTY`
-> if character has no party: log error and return.

No member was ever added.

A fresh `CDungeon` starts with `deadEvent = nullptr`. Automatic empty-dungeon destruction is scheduled only from `DecMember` when the tracked character set transitions to empty. Since this rejected dungeon never had a member, no dead event is scheduled.

Result: the newly-created private map/dungeon remains registered without an automatic cleanup trigger.

A second active API has the same ordering defect:
`d.new_jump_guild`
creates the dungeon before checking whether the current character has a guild. A non-guild caller can therefore produce the same orphan state.

Current mapped standard dungeon quests mostly use `d.new_jump_party` / `d.new_jump_all`, so gameplay frequency is script-dependent; the registered Lua API failure path itself is deterministic.


### BUG-DUNGEON-002 — SpawnMoveUnique continues after successful spawn
- Statik durum: **doğrulandı**
- Sınıf: spawn control flow / untracked entities / resource amplification
- Surface: registered Lua `d.spawn_move_unique`

`CDungeon::SpawnMoveUnique` runs a 100-iteration spawn-attempt loop but does not `break` or `return` after a successful spawn.

Every successful iteration creates a new mob, applies the unique affect, binds it to the dungeon and starts movement toward the target area.

The registry update uses:
`m_map_UniqueMob.insert(key, ch)`.

Because the map accepts only one value for a key, the first success is tracked while later successful mobs under the same requested key are not inserted. A single valid invocation can therefore create up to 100 entities while only one is reachable through the unique registry.

This is deterministic whenever repeated spawn attempts succeed.

### BUG-DUNGEON-003 — multiple unique keys for one character can leave a dangling pointer
- Statik durum: **doğrulandı**
- Sınıf: raw-pointer lifecycle / use-after-free
- Surface: registered Lua `d.set_unique`

`SetUnique(key, vid)` allows one `LPCHARACTER` to be stored under multiple distinct keys.

`DeadCharacter(ch)` removes only the first matching map entry and then breaks.

Deterministic direct-destruction example:
1. bind mob X as key A;
2. bind the same VID as key B;
3. execute a generic dungeon purge;
4. `DestroyCharacter(X)` invokes `DeadCharacter(X)` once;
5. one alias is erased;
6. X is deleted;
7. the second alias remains in `m_map_UniqueMob`.

Normal death gives two cleanup passes (death + later manager destruction), so three or more aliases still leave a stale entry.

Subsequent unique APIs dereference that stale raw pointer, including `GetUniqueVid`, `IsUniqueDead`, `GetUniqueHpPerc` and `UniqueSet*`.



### BUG-DUNGEON-004 — duplicate unique key leaves spawned/marked entity outside unique registry
- Statik durum: **doğrulandı**
- Sınıf: unique registry consistency / untracked entity
- Surfaces: `d.spawn_unique`, `d.spawn_move_unique`, `d.set_unique`

Unique registration uses `std::map::insert` and ignores the boolean insertion result.

If a key already exists, `SpawnUnique` can still spawn a new mob, set its dungeon pointer and apply the dungeon-unique affect even though the map remains bound to the older mob.

Similarly, `SetUnique` with an already-used key and another VID leaves the old mapping unchanged while still applying the unique affect to the new character.

The registry and actual dungeon entities therefore diverge deterministically under duplicate-key use. Later `get/kill/purge/unique_set*` operations address only the first registered character.
