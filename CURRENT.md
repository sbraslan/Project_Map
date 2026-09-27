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
- Sung Mahi tower maps are explicitly treated as SungMa maps by `CHARACTER::IsSungmaMap()`.
- Tower SungMa STR/HP/MOVE/IMMUNE requirements override normal map data via `GetSungMahiTowerDungeonValue()`.
- The tower value table is hard-coded as 4 x 51 and indexed by dungeon flag `dungeonLevel`; HIT_PCT is forced to 0 in the tower.
- No bounds check exists at the local lookup site; bug promotion is deferred until all `dungeonLevel` writers are traced and range guarantees are proven.
- Base Conqueror/SungMa player progression is persisted in the normal player table and restored on character load.

## Exact next work
1. Find every writer of dungeon flag `dungeonLevel` and prove its runtime range (0..50).
2. Resolve server/quest producers for the nine Sung Mahi client command strings and the source of `sungMahiQuest`.
3. Trace quest-button entry into the tower instance.
4. Map tower-specific completion/rank/reward persistence.
5. Only then promote any verified bug.

GitHub state is canonical.
