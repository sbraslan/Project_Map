# Sung Mahi Tower — Verified Bugs

**Phase:** Detection / Mapping Only
**Date:** 2026-09-27

## BUG-SMT-001 — Tower quest runtime implementation is missing from the tracked Project_Game quest set

**Status:** VERIFIED — STATIC / REPOSITORY INTEGRATION  
**Scope:** Project_Game quest packaging / Sung Mahi Tower entry and progression

### Evidence
- `ENABLE_SUNG_MAHI_TOWER` is enabled server-side.
- `MAP_SMG_DUNGEON_01` (386) and `MAP_SMG_DUNGEON_02` (387) are both loaded only on `game-ch99-core99`.
- Map 386 spawns NPC vnum `4020`, labelled `Sung Mahis Höllenturm`.
- Client entry requires a server-provided quest index:
  `uisungmahi.py -> event.QuestButtonClick(constInfo.sungMahiQuest)`.
- `event.QuestButtonClick()` sends `HEADER_CG_SCRIPT_BUTTON`; the server routes it through `CInputMain::ScriptButton()` to `CQuestManager::QuestButton()`, which resolves a real `QUEST_BUTTON_EVENT` quest index.
- `constInfo.sungMahiQuest` defaults to `0` and is only useful after the server producer supplies `SetSungMahiQuest`.
- The server exposes Sung Mahi-specific Lua APIs expected to be used by quest logic:
  `d.set_dungeon_difficulty`, `d.spawn_mob_dir_nomove`, `d.set_unique_master`, `d.clear_dungeon_flags`, and `pc.sung_mahi_curse_hp`.
- The tracked quest source set contains no Sung Mahi/SMH tower quest source, the active `quest_list` contains no such quest, and the compiled `quest/object/state` set contains no Sung Mahi tower state.
- No `quest/object/4020/` handler exists for the tower NPC.
- The visible dungeon quest sources do not call the Sung Mahi-specific Lua APIs.

### Impact
Within this repository snapshot, the quest-driven producer needed to:
- open/populate the tower UI;
- assign `sungMahiQuest`;
- process the quest-button entry action;
- set/synchronize tower floor state;
- drive tower-specific `cmdchat` updates;
- orchestrate the exposed Sung Mahi dungeon Lua APIs

is absent from the tracked runtime quest set.

That leaves the mapped tower feature incomplete/non-operational from the repository contents alone. In particular, NPC 4020 has no compiled quest handler in the tracked `object` tree, and the client entry path depends on a quest index that defaults to zero until a producer sets it.

### Boundary / caveat
This bug is verified against the tracked repository snapshot. If the live server installs an additional Sung Mahi quest/object package from outside `Project_Game`, runtime behavior can differ. Such an external package is not present in the canonical source snapshot and therefore cannot currently be audited.

### No source change
No production source, Python, quest, map, config, or game data was modified. This record is detection-only.
