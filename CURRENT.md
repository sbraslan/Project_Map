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
- Maps 386/387 are loaded only on `game-ch99-core99`; the Sung Mahi monthly reward timer is also bound to that same host.
- Map 386 contains tower NPC vnum 4020; map 387 is the private tower instance map.
- Quest-button entry is fully traced: Python `QuestButtonClick` -> `HEADER_CG_SCRIPT_BUTTON` -> `CInputMain::ScriptButton` -> `CQuestManager::QuestButton` -> `QUEST_BUTTON_EVENT`.
- Tower-specific Lua APIs exist, including `d.set_dungeon_difficulty`, but the tracked quest package contains no Sung Mahi tower quest source, quest_list entry, object/state entry, or object/4020 handler.
- `m_bDungeon_Difficulty` drives mob scaling, while dungeon flag `dungeonLevel` drives player SungMa requirements; C++ does not automatically synchronize them.
- Verified `BUG-SMT-001`: tracked Project_Game quest runtime implementation for Sung Mahi Tower is missing.

## Exact next work
1. Find every `sung_mahi_ranking` row writer and close ranking/completion persistence.
2. Audit `m_bDungeon_Difficulty` vs `dungeonLevel` synchronization risk.
3. Trace tower room/monster progression hooks and unique-master kill behavior.
4. Revisit visibility-gated live client commands after server timing is mapped.
5. No production source changes.

GitHub state is canonical.
