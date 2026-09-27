# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Sung Mahi Tower
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/sung_mahi_tower.md`
**Verified bug registry:** `bugs/sung_mahi_tower.md`
**Last updated:** 2026-09-27

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Just mapped
- The monthly Sung Mahi ranking path is a reader/resetter: it selects fastest players from `sung_mahi_ranking` and later truncates the table.
- Generic Lua `mysql_direct_query` is available, but no dedicated mapped C++ Sung Mahi ranking writer was found; the missing tower quest package remains the likely INSERT/UPDATE producer.
- Verified `BUG-SMT-002`: monthly reset explicitly loads `Questlibs/dungeonInfoLibrary.lua`, but that file is absent from the tracked Project_Game tree.
- Tower room activation hook is mapped: a unique-master hit triggers `AggregateMonsterByMaster()`, removes NOMOVE/NOATTACK from all monsters in the private instance map, and sets `chessWrongMonster=1`.
- Verified bugs: `BUG-SMT-001`, `BUG-SMT-002`.

## Exact next work
1. Audit `m_bDungeon_Difficulty` vs dungeon flag `dungeonLevel` for divergence paths.
2. Trace remaining room-clear/kill/completion and reward issuance hooks.
3. Compare client live-command visibility gating with map-load timing.
4. Treat ranking row production as missing-quest responsibility unless another writer is found.
5. No production source changes.

GitHub state is canonical.
