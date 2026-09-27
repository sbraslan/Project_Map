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

## Client command transport
`Project_Binary/root/game.py` registers nine Sung Mahi names in the normal server-command dispatcher (`stringCommander.Analyzer`):
- `ClearSungMahiInfo`
- `SetSungMahiQuest`
- `UpdateSungMahiInfo`
- `OpenSungMahiWindow`
- `UpdateSungMahiNotice`
- `SungMahiClearNotice`
- `UpdateRoomLevel`
- `UpdateTowerLevel`
- `UpdateRoomTime`

Client-side consumers are now mapped:
- `ClearSungMahiInfo` clears `constInfo.sungMahiInfo`, `sungMahiLevelInfo`, and `sungMahiQuest`.
- `SetSungMahiQuest` stores the quest-button index used for tower entry.
- `UpdateSungMahiInfo` appends floor/rank/time/completion tuples and updates the current cleared-floor count.
- `OpenSungMahiWindow` toggles the tower entry window.
- notice/room/tower/time commands are forwarded through `Interface` to the Sung Mahi minimap cover.

The exact server-side producers for these command strings remain unresolved and are the next server/quest trace target.

## Entry window lifecycle
`Project_Binary/root/uisungmahi.py`:
- builds the floor list and displays rank, clear time, element and configured rewards from `constInfo` caches;
- `Open()` calls `SetCompletedDungeons(constInfo.sungMahiLevelInfo)`;
- the selected floor is tracked as a zero-based `previousListItem`;
- entry confirmation only proceeds when `constInfo.sungMahiLevelInfo == previousListItem`;
- accepted entry invokes `event.QuestButtonClick(int(constInfo.sungMahiQuest))` and closes the UI.

Therefore the client does not directly send a dedicated Sung Mahi entry packet from this window. Entry is routed through the quest-button mechanism using the server-provided quest index.

## In-tower information-board lifecycle
`Project_Binary/root/interfacemodule.py` forwards live tower commands only while `wndMiniMap.sungMahiCover` is visible:
- `UpdateSungMahiNotice` -> description/curse text;
- `SungMahiClearNotice` -> clears notice state;
- `UpdateRoomLevel` -> room-stage indicators;
- `UpdateTowerLevel` -> floor label;
- `UpdateRoomTime` -> countdown gauge.

`Project_Binary/root/uiminimap.py`:
- constructs `SungMahiCover` for the minimap;
- automatically shows it only on `metin2_map_smhdungeon_02`;
- `SetRoomLevel` resets the room timer gauge when stage changes;
- `UpdateRoomTime` stores the supplied time and a local global timestamp for countdown updates;
- exit confirmation uses the generic `/restart_here` chat command.

This establishes two distinct client surfaces:
1. tower entry/progression window (`uisungmahi.py`);
2. live in-tower board (`uiminimap.py::SungMahiCover`).

## Initial roots
### Client
- `Project_Binary/root/game.py` — Sung Mahi server-command callbacks and dispatcher registration.
- `Project_Binary/root/uisungmahi.py` — entry/progression controller.
- `Project_Binary/root/interfacemodule.py` — command forwarding to entry window / in-tower overlay.
- `Project_Binary/root/uiminimap.py::SungMahiCover` — in-tower room/floor/timer/notice board.
- `Project_Binary/root/uiscript/sungmaheetowerenter.py` — entry window.
- `Project_Binary/root/uiscript/sungmaheetowerinformationboard.py` — in-tower information board.
- `Project_Binary/root/constinfo.py` — `sungMahiInfo`, `sungMahiLevelInfo`, `sungMahiQuest`, reward/element caches.
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
1. Resolve the server/quest producers for the nine server-command strings and the source of `sungMahiQuest`.
2. Trace the quest-button entry path into the tower instance.
3. Map SungMa attribute loading and `IsSungmaMap()/GetSungmaMapAttribute()`.
4. Map Conqueror/SungMa persistence fields and tower reward persistence.
5. Check visibility-gated live command handling only after producer timing is known; do not register a bug before end-to-end closure.
