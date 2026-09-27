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


## SungMa map/tower attribute resolution
Server-side attribute flow is now partially closed in `game/src/char.cpp`.

`CHARACTER::IsSungmaMap()` returns true for normal SungMa-table maps and also explicitly for the Sung Mahi dungeon via `IsSungMahiDungeon(GetMapIndex())`.

`CHARACTER::GetSungmaMapAttribute(point)` normally reads `g_map_SungmaTable[mapIndex]`, but Sung Mahi Tower overrides four requirements when inside the tower:
- STR -> `GetSungMahiTowerDungeonValue(0)`
- HP -> `GetSungMahiTowerDungeonValue(1)`
- MOVE -> `GetSungMahiTowerDungeonValue(2)`
- IMMUNE -> `GetSungMahiTowerDungeonValue(3)`
- HIT_PCT is forced to `0` for the Sung Mahi dungeon.

`CHARACTER::GetSungMahiTowerDungeonValue()` contains a hard-coded `4 x 51` table. It obtains the active floor from the current dungeon instance flag named `"dungeonLevel"` and indexes the table with that value. Therefore the runtime SungMa requirements for the tower are driven by the dungeon flag, not by the normal map `sungma_attr.txt` entry.

The table supports indices 0..50. No local bounds check is present in `GetSungMahiTowerDungeonValue()`; bug status is intentionally deferred until every producer of `dungeonLevel` is traced and its range guarantee is verified.

## Conqueror/SungMa persistence roots
`game/src/char.cpp` confirms the generic Yohara player state is persisted through the normal player table:
- `conqueror_level`
- `conqueror_level_step`
- `conqueror_exp`
- `conqueror_st` -> `POINT_SUNGMA_STR`
- `conqueror_ht` -> `POINT_SUNGMA_HP`
- `conqueror_mov` -> `POINT_SUNGMA_MOVE`
- `conqueror_imu` -> `POINT_SUNGMA_IMMUNE`
- `conqueror_point`

Load restores these fields into real/current character points. This establishes that base Conqueror/SungMa character progression is persistent independently of tower UI state.


## `dungeonLevel` writer investigation
The consumer-side range is now clearer:
- `CHARACTER::SetDungeonMultipliers(uint8_t dungeonLevel)` explicitly rejects values below 1 or above 50.
- `GetSungMahiTowerDungeonValue()` uses the dungeon flag `"dungeonLevel"` directly as an index into a 0..50 table, but does not repeat the same bounds guard locally.
- `CHARACTER::IsSungMahiDungeon(long)` is defined in `char.h` as the private-instance range belonging to `MAP_SMG_DUNGEON_02`; therefore this lookup is scoped to Sung Mahi Tower dungeon instances, not arbitrary Yohara maps.

A direct writer for dungeon flag `"dungeonLevel"` was not found in the inspected C++ paths. Generic Lua dungeon flags are writable through `d.setf(...)` / `CDungeon::SetFlag()`, so the likely producer is quest-side tower logic.

Repository state observation:
- the active `quest_list` contains no Sung Mahi / SMH tower quest source entry;
- no quest source path whose filename contains Sung/Mahi/SMH exists in the checked quest tree;
- therefore the exact quest-side producer cannot yet be proven from the visible source set. This is a mapping gap, not yet a bug.

Do not promote the missing local bounds check to a verified bug until the actual tower quest/runtime producer or an equivalent range guarantee is recovered.

## Ranking / monthly reward persistence root
`game/src/questmanager.cpp` contains a dedicated monthly Sung Mahi reward event:
- iterates tower levels `1..SUNG_MAHI_MAX_LEVEL`;
- queries `sung_mahi_ranking` for the fastest player per floor;
- awards item `50249` via mailbox or `item_award`;
- truncates `sung_mahi_ranking` after monthly payout;
- stores month state in event flag `sungMahiLastMonth`.

This proves tower ranking persistence exists in a dedicated SQL table and is seasonally reset independently of generic Conqueror player progression.


## Canonical tower level limit
`common/length.h::ESungMahiDungeon` defines:
- `SUNG_MAHI_MAX_LEVEL = 50`
- damage multiplier = 5
- defence multiplier = 5
- HP multiplier = 250

This matches both:
- the `GetSungMahiTowerDungeonValue()` 0..50 lookup table;
- the `SetDungeonMultipliers()` explicit 1..50 guard;
- the monthly ranking loop in `questmanager.cpp`, which iterates levels 1 through `SUNG_MAHI_MAX_LEVEL`.

Therefore 50 is the canonical server-side maximum tower level. The remaining unresolved question is not the intended range, but whether every runtime writer of dungeon flag `dungeonLevel` enforces that intended range before `GetSungMahiTowerDungeonValue()` consumes it.

## Additional server behavior roots
`game/src/char.cpp` also treats both `MAP_SMG_DUNGEON_01` and `MAP_SMG_DUNGEON_02` as special dungeon maps in generic map restrictions. Character state includes tower-specific no-move/no-attack/unique-master support under `ENABLE_SUNG_MAHI_TOWER`.

This indicates the missing tower runtime logic likely orchestrates generic character flags and dungeon flags rather than living in a dedicated `SungMahi*.cpp` manager.


## Quest runtime integration closure
The quest-button transport is now end-to-end mapped:
`uisungmahi.py -> event.QuestButtonClick(index) -> CPythonNetworkStream::SendScriptButtonPacket() -> HEADER_CG_SCRIPT_BUTTON -> CInputMain::ScriptButton() -> CQuestManager::QuestButton() -> NPC::OnButton(..., QUEST_BUTTON_EVENT)`.

Therefore `sungMahiQuest` must be a real loaded quest index with a button event handler.

The server also exposes tower-specific Lua APIs:
- `d.set_dungeon_difficulty`
- `d.spawn_mob_dir_nomove`
- `d.set_unique_master`
- `d.clear_dungeon_flags`
- `pc.sung_mahi_curse_hp`

`d.set_dungeon_difficulty` writes `CDungeon::m_bDungeon_Difficulty`. Group-spawned mobs then read that value in `CHARACTER_MANAGER::SpawnGroup()` and apply `SetDungeonMultipliers()`.

This is separate from dungeon flag `"dungeonLevel"`, which drives `GetSungMahiTowerDungeonValue()`. The two tower level states are not automatically linked in C++; quest logic must keep them synchronized.

## Channel/map placement
`Project_Game` channel configs show:
- maps 386 and 387 are absent from normal ch1/ch2 cores;
- both are loaded on `game-ch99-core99`;
- the monthly Sung Mahi reward timer is also explicitly created only on hostname `game-ch99-core99`.

Map 386 (`metin2_map_smhdungeon_01`) contains:
- vnum 4020 — `Sung Mahis Höllenturm`
- vnum 10126 — exit
- mailbox and merchant NPCs.

Map 387 is the private Sung Mahi tower map used by `IsSungMahiDungeon()`.

## Repository quest-package gap
The quest package was checked at source, active-list, and compiled-object levels:
- no Sung Mahi/SMH tower quest exists in `quest_list`;
- no Sung Mahi/SMH tower quest source exists among the tracked `.quest` files;
- no tower state exists in `quest/object/state`;
- no `quest/object/4020/` handler exists for the tower NPC;
- the visible dungeon quest sources do not call the tower-specific Lua APIs.

This promotes the prior mapping gap to verified repository-integration bug `BUG-SMT-001`: the enabled maps/client/server hooks have no tracked quest runtime implementation to drive entry, floor state, command producers, or the tower Lua API orchestration.

Caveat: an untracked external quest package installed only on a live deployment could change runtime behavior; such a package is absent from this repository snapshot.

## Verified bugs
- `BUG-SMT-001` — Sung Mahi Tower quest runtime implementation is missing from the tracked Project_Game quest package.

## Exact next work
1. Map all remaining server-side Sung Mahi persistence/reward SQL paths and determine where `sung_mahi_ranking` rows are written.
2. Audit the two unsynchronized level states: `m_bDungeon_Difficulty` versus dungeon flag `dungeonLevel`.
3. Trace tower-specific monster/room progression hooks (`unique master`, no-move/no-attack, kill handling).
4. Revisit the visibility-gated client live commands after server timing is mapped.
5. Keep missing quest runtime as BUG-SMT-001; do not patch source during detection phase.
