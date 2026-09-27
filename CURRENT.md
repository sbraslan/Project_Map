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
- All nine Sung Mahi names are client server-command callbacks registered in `game.py`.
- Entry UI does not send a dedicated tower packet; it calls `event.QuestButtonClick(constInfo.sungMahiQuest)`.
- Entry/progression UI and live in-tower minimap board are separate client surfaces.
- Live room/floor/time/notice updates are forwarded through `Interface` only while `sungMahiCover` is visible.
- `sungMahiCover` is shown for `metin2_map_smhdungeon_02`; exit uses generic `/restart_here`.

## Exact next work
1. Resolve server/quest producers for the nine client command strings and the source of `sungMahiQuest`.
2. Trace quest-button entry into the tower instance.
3. Trace `IsSungmaMap()/GetSungmaMapAttribute()` loading/enforcement.
4. Map Conqueror/tower persistence and reward boundaries.
5. Only then promote any end-to-end verified bug.

GitHub state is canonical.
