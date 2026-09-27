# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Sung Mahi Tower
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/sung_mahi_tower.md`
**Last updated:** 2026-09-27

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Just mapped
- `IsSungMahiDungeon()` is scoped to private instances of `MAP_SMG_DUNGEON_02`.
- `SetDungeonMultipliers()` accepts only dungeon levels 1..50.
- `GetSungMahiTowerDungeonValue()` still indexes its 0..50 table directly from dungeon flag `dungeonLevel` without a local bounds check.
- No C++ writer for `dungeonLevel` was found; generic Lua dungeon flags are writable through `d.setf/CDungeon::SetFlag`.
- The visible `quest_list` and quest source tree contain no Sung Mahi/SMH tower quest source entry, so the exact producer remains unresolved from the current source set.
- `questmanager.cpp` confirms dedicated SQL ranking persistence in `sung_mahi_ranking`, monthly fastest-per-floor rewards, then table truncation/reset.

## Exact next work
1. Recover the quest/runtime producer for `dungeonLevel` and prove its 1..50 guarantee.
2. Resolve producers for the nine Sung Mahi server-command strings and `sungMahiQuest`.
3. Trace quest-button entry into the private tower instance.
4. Continue ranking/completion/reward SQL boundary mapping.
5. Only then promote any verified bug.

GitHub state is canonical.
