# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Dungeon Core
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/dungeon_core.md`
**Last updated:** 2026-09-26

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Dungeon Core checkpoint
Verified bugs:
- `BUG-DUNGEON-001` — rejected `d.join` / `d.new_jump_guild` entry can orphan an empty private dungeon without dead-event cleanup.

Newly closed:
- party/dungeon member counters and raw party-key cleanup;
- destination SetDungeon binding;
- participant registry uses PID/name, not raw character pointers;
- unique mob raw pointers are erased on normal death and manager destruction paths;
- all 98 source quest files checked for suspicious kill/potion/revive getters: no current callers;
- Devil Catacomb item-group path adds live reachability to existing `BUG-PARTY-001`.

Not promoted:
- eliminate event null-ordering: destructor cancellation still closes normal stale-event path;
- nested JumpParty ownership: semantic weakness, no mapped live nested caller;
- item-group exchange risk: specific Reaper's Credit item anti-flags unavailable.

## Exact next work
1. Audit regen lifetime.
2. Audit bulk purge/kill pending-destroy behavior.
3. Audit unique alias edge cases.
4. Revisit nested JumpParty reachability.
5. Continue Dungeon Core closure.

GitHub state is canonical.
