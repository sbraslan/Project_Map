# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Aura System Static Mapping  
**Status:** STATIC MAPPING IN PROGRESS / 3 VERIFIED STATIC BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Aura System  
**System:** `systems/aura.md`  
**Bugs:** `bugs/aura.md`  
**Tests:** `tests/aura.md`  
**Last completed subsystem:** Dragon Soul / Alchemy  
**Effective completed/readiness-covered subsystems:** 24  
**First future live gate:** `DUNGEON-T10`  
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime, crafted-packet, crash, sanitizer and fault-injection execution remains locked until an explicit phase change.

## Dragon Soul / Alchemy closure

Dragon Soul is now **STATIC COMPLETE**.

Canonical verified findings:
`BUG-DS-001..BUG-DS-011`.

Closure added:
- `BUG-DS-009` — relog with persisted DS set can execute set cleanup while active deck is still -1; uint8 arithmetic wraps the current build's start index to wear 27 and can subtract DS-set values from ordinary late equipment.
- `BUG-DS-010` — `DSManager::PullOut()` can destroy a count-1 extractor and later dereference it in success/failure log formatting.
- `BUG-DS-011` — active daily-gift event with `ds_dg_id=0` can skip level/qualification checks for a never-participated character whose quest event_id is also 0.

Cross-window aliasing produced no additional DS-specific promoted defect. Malformed grade/step boundaries remain unpromoted because current tracked data does not establish malformed reachability.

Deferred ownership:
`DS-T01..DS-T11`, none executed.

Current effective readiness coverage is **24/24 STATIC COMPLETE subsystems**, plus folded Guild lifecycle.

## Active Aura System state

Verified static bugs:
- `BUG-AURA-001` — Aura's intended opener-distance gate is bypassed after the window opens. `IsAuraRefineWindowCanRefine()` first calls generic `CanHandleItem()`, which rejects the Aura window itself; check-in/check-out/accept then ignore that false result whenever Aura is open and opener is non-null.
- `BUG-AURA-002` — EVOLVE validates required material through total `CountSpecifyItem()` quantity but consumes only the checked-in SUB stack, allowing underpayment with split stacks through a crafted client.
- `BUG-AURA-003` — Aura Eraser never validates COSTUME_AURA and can zero generic socket 2 on an unrelated unequipped item.

Deferred tests:
- `AURA-T01..AURA-T03`, none executed.

## Exact next work
1. audit booster timer/equip/unequip persistence and absorption-rate application;
2. close forced-warp/disconnect/destructor locked-item cleanup;
3. audit opener lifetime and cross-window coexistence;
4. verify wedding-item absorption policy against tracked data;
5. map Aura default socket initialization/refine chains;
6. close Aura visual/proto/client persistence surfaces;
7. consolidate Aura runtime ownership and decide STATIC COMPLETE.

Do not execute `AURA-T01..AURA-T03` or any other runtime test.
The global future runtime order remains locked with `DUNGEON-T10` first.

GitHub state is canonical.
