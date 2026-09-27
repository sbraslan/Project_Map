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
- `common/length.h::ESungMahiDungeon` proves the canonical tower maximum is exactly 50.
- The same 50-level boundary matches the 4x51 SungMa lookup table, `SetDungeonMultipliers()` 1..50 guard, and monthly ranking loop.
- The unresolved risk is therefore specifically whether every runtime writer of dungeon flag `dungeonLevel` enforces 1..50 before the unguarded table lookup.
- Generic server code also recognizes both `MAP_SMG_DUNGEON_01` and `MAP_SMG_DUNGEON_02` as special dungeon maps and exposes tower-specific character control flags.

## Exact next work
1. Recover the missing tower quest/runtime writer for `dungeonLevel`.
2. Resolve the nine `cmdchat` producers and `sungMahiQuest`.
3. Trace entry flow across `MAP_SMG_DUNGEON_01` and private `MAP_SMG_DUNGEON_02`.
4. Continue ranking/completion/reward SQL boundary mapping.
5. Only then promote any verified bug.

GitHub state is canonical.
