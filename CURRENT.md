# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Static Mapping Resumed
**Active subsystem:** Costume / Appearance / ChangeLook
**Status:** STATIC MAPPING IN PROGRESS
**Machine state:** `STATE.json`
**Item automation:** `ITEM_CREATION_AUTOMATION.md`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime execution is still locked.

## Side workflow saved
- Trigger phrase: `item ekleyeceğiz`.
- Canonical generation rules: `ITEM_CREATION_AUTOMATION.md`.
- Reference VNUM -> nearest structural template -> proposed VNUM/ShapeIndex -> item_proto + companion MSM/item files.
- No source/game write is authorized by this workflow.

## Active mapping
New subsystem opened: **Costume / Appearance / ChangeLook**.

Next:
1. map server costume equip/change-look state;
2. map client armor/shape resolution;
3. map proto/MSM/item_list dependencies;
4. record bugs/candidates only when statically verified.

GitHub state is canonical.
