# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active subsystem:** Zodiac Temple / 12ZI  
**Status:** STATIC MAPPING OPEN / 0 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Navigation:** `INDEX.md`  
**System map:** `systems/zodiac_temple.md`  
**Bug registry:** `bugs/zodiac_temple.md`  
**Deferred tests:** `tests/zodiac_temple.md`  
**Previous completed subsystem:** 6th/7th Attribute — STATIC COMPLETE / 9 VERIFIED BUGS  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Why Zodiac is active
Global feature discovery found `ENABLE_12ZI` with dedicated, currently unrepresented roots:
- `game/src/zodiac_temple.cpp/.h`;
- `game/src/char_zodiac_temple.cpp`;
- `game/src/char_battle_zodiac.cpp`;
- `game/src/questlua_zodiac_temple.cpp`;
- `Project_Binary/root/ui12zi.py`;
- `Project_Game/share/data/dungeon/zodiac/` and `metin2_12zi_stage`.

## Exact resume cursor
1. Verify deployment/quest entry and portal argument validation.
2. Map private-instance + party/member + reconnect/logout/destruction lifecycle.
3. Audit player-accessible Zodiac chat commands and reward state.
4. Audit combat event raw-pointer lifetime and map destruction.
5. Map client/server parity, special effect packet and UI command paths.
6. Promote only source-proven findings; keep runtime locked.

## Mapping acceleration
`index/features.json` is being refreshed to include the latest closed maps and Zodiac. Context7 remains supplementary; Metin2 source is authoritative.
