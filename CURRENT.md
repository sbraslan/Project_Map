# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Dungeon Core
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/dungeon_core.md`
**Last updated:** 2026-09-27

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Dungeon Core checkpoint
Verified bugs:
- `BUG-DUNGEON-001` — rejected `d.join` / `d.new_jump_guild` can orphan an empty private dungeon.
- `BUG-DUNGEON-002` — `SpawnMoveUnique` does not stop after success and can create up to 100 mobs while tracking one key.
- `BUG-DUNGEON-003` — multi-key `SetUnique` aliases can survive character destruction as dangling raw pointers.

Closed without new bug:
- persistent regen event/REGEN lifetime: event cancellation + dungeon-ID lookup + pointer/id validity guard closes the mapped UAF path;
- bulk `KillAll/Purge/KillMonsters` traversal: `SECTREE_MAP::for_each` uses an entity snapshot, avoiding direct iterator invalidation;
- normal participant and party-member bookkeeping previously closed.

Still candidate / not promoted:
- nested `JumpParty` can bypass exclusive-dungeon equality when party already has an ownership pointer, but current nested quest reachability is not yet established;
- eliminate event null-ordering remains incorrect but no surviving-event lifecycle path is mapped;
- lower-level stale SECTREE relationship erase has no established normal producer.

## Exact next work
1. Scan current source quests for `d.spawn_move_unique`, multi-key `d.set_unique`, and nested `d.new_jump_party` reachability.
2. Audit duplicate-key behavior in `SpawnUnique/SetUnique`.
3. Audit dungeon manager ID wrap against event identity.
4. Close eliminate-event null ordering.
5. Decide Dungeon Core STATIC COMPLETE.

GitHub state is canonical.
