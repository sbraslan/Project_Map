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
