# Sung Mahi Tower

**Status:** PARTIAL — ACTIVE
**Phase:** Detection / Mapping Only
**Date:** 2026-09-27

> Source/game repositories are read-only. Only Project_Map is writable.

## Feature/config
`ENABLE_SUNG_MAHI_TOWER` is enabled in `common/CommonDefines.h`.

## Initial scope
Map the full Sung Mahi Tower path across:
- server manager / dungeon lifecycle;
- packet and command transport;
- player eligibility / SungMa attributes;
- room/tower progression;
- rewards and persistence;
- client Python/UI state;
- quest/config/data integration;
- multi-core/channel behavior.

## Initial roots
### Client
- `Project_Binary/root/game.py` — Sung Mahi server-command callbacks:
  - `ClearSungMahiInfo`
  - `SetSungMahiQuest`
  - `UpdateSungMahiInfo`
  - `OpenSungMahiWindow`
  - `UpdateSungMahiNotice`
  - `SungMahiClearNotice`
  - `UpdateRoomLevel`
  - `UpdateTowerLevel`
  - `UpdateRoomTime`
- `Project_Binary/root/uisungmahi.py` — main Sung Mahi UI/controller.
- `Project_Binary/root/uiscript/sungmaheetowerenter.py` — entry window.
- `Project_Binary/root/uiscript/sungmaheetowerinformationboard.py` — in-tower information board.
- `Project_Binary/root/constinfo.py` — `sungMahiInfo`, `sungMahiLevelInfo`, `sungMahiQuest` client cache.
- locale data:
  - `locale/locale/common/sungmahee_tower/standard/sungmahee_tower_element.txt`
  - `locale/locale/common/sungmahee_tower/standard/sungmahee_tower_reward.txt`

### Server / generic Yohara integration
No dedicated `SungMahi*.cpp` file was found by filename. Tower/Yohara behavior is embedded in generic character/battle/map paths.

Known roots:
- `game/src/char_battle.cpp` — SungMa map combat restrictions:
  - player damage on SungMa maps is halved when `POINT_SUNGMA_STR` is below the map requirement;
  - non-Conqueror characters deal zero damage to non-PC targets on SungMa maps;
  - precision/block logic uses the SungMa map attribute `POINT_HIT_PCT`.
- `game/src/char.cpp/.h` — SungMa map/attribute and conqueror-player state roots (next sweep).
- `common/tables.h` / player persistence — Conqueror/SungMa player fields (next sweep).
- map data in Project_Game uses `sungma_attr.txt` across Yohara maps.
- Sung Mahi monster data exists under `share/data/monster/smhtower_*`.

### Data/map roots
- `Project_Game/share/data/monster/smhtower_boss`
- `Project_Game/share/data/monster/smhtower_general`
- `Project_Game/share/data/monster/smhtower_king`
- `Project_Game/share/data/monster/smhtower_knight`
- `Project_Game/share/data/monster/smhtower_magic`
- `Project_Game/share/data/monster/smhtower_soldier*`
- SungMa attribute files are present on multiple Yohara maps via `sungma_attr.txt`.

## Verified bugs
None yet.

## Exact next work
1. Trace the server producers of the nine Sung Mahi client commands.
2. Map `uisungmahi.py` entry/reward/room/tower state transitions.
3. Map SungMa attribute loading and `IsSungmaMap()/GetSungmaMapAttribute()`.
4. Map Conqueror/SungMa persistence fields and tower reward persistence.
5. Begin verified bug detection only after end-to-end flow is established.
