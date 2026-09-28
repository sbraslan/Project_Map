# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Dragon Soul / Alchemy Static Mapping
**Status:** STATIC MAPPING IN PROGRESS / 7 VERIFIED STATIC BUGS / EXECUTION LOCKED
**Machine state:** `STATE.json`
**Active subsystem:** Dragon Soul / Alchemy
**System:** `systems/dragon_soul.md`
**Bugs:** `bugs/dragon_soul.md`
**Tests:** `tests/dragon_soul.md`
**Last completed subsystem:** Acce / Sash
**Effective completed/readiness-covered subsystems:** 23
**First future live gate:** `DUNGEON-T10`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime, crafted-packet, crash, sanitizer and fault-injection execution remains locked until an explicit phase change.

## Acce / Sash closure

Acce is now **STATIC COMPLETE**.

Canonical verified findings:
`BUG-ACCE-001..008`.

Newest closure findings:
- `BUG-ACCE-007` — Acce open state can survive warp because `CanWarp()` omits `W_ACCE` and `WarpSet()` does not force `AcceClose()`; stale flags can keep `CanHandleItem()` blocked after arrival.
- `BUG-ACCE-008` — reversal clears socket0 and normal attributes but never clears copied element/set metadata; those fields are persisted and can remain visible after full refresh/relog.

Visual/data dependency is closed:
- sash VNUM -> `item_list.txt` WING -> GR2;
- `item_scale.txt` -> per-job/per-sex scale through `CItemManager::LoadItemScale()`;
- equipped visual -> `PART_ACCE` -> `Bip01 Spine2`.

Combine/refine-chain and normal uint8 inventory-cell boundaries were closed without another promoted bug.

Deferred ownership:
`ACCE-T01..ACCE-T08`, none executed.

Current effective readiness coverage is **23/23 STATIC COMPLETE subsystems**, plus folded Guild lifecycle.

## Active Dragon Soul / Alchemy state

Verified static bugs:
- `BUG-DS-001` — stale DS set contribution after breaking an active complete set.
- `BUG-DS-002` — Dragon Heart extraction logs a source pointer after count-1 destruction.
- `BUG-DS-003` — successful strength refine can leave an ownerless zero-count CItem registered in memory.
- `BUG-DS-004` — Change Attribute server path accepts ordinary strength-refine materials.
- `BUG-DS-005` — RefineStep table validation checks the wrong table node; dormant with current tracked data.
- `BUG-DS-006` — any open DS refine opener token authorizes Change Attribute packets; mode is not server-bound.
- `BUG-DS-007` — DS refine opener can survive warp and keep item handling locked.

Current server/client Dragon Soul table files are mapped as matching for the tracked deployment. Candidate malformed-data boundaries remain unpromoted.

## Exact next work
1. close grade/step/strength material-count and stack semantics;
2. inspect DS deck/set reactivation and relog persistence;
3. close refine-window overlap and cross-window interactions;
4. inspect extraction tool/source aliasing;
5. inspect qualification/daily quest lifecycle;
6. only then decide Dragon Soul STATIC COMPLETE/readiness promotion.

Do not execute `DS-T01..DS-T07`.
The global future runtime order remains locked with `DUNGEON-T10` first.

GitHub state is canonical.
