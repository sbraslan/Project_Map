# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Acce / Sash Static Mapping
**Status:** STATIC MAPPING IN PROGRESS / 7 VERIFIED STATIC BUGS / EXECUTION LOCKED
**Machine state:** `STATE.json`
**Active subsystem:** Acce / Sash
**System:** `systems/acce.md`
**Bugs:** `bugs/acce.md`
**Tests:** `tests/acce.md`
**First future live gate:** `DUNGEON-T10`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/crafted-packet/sanitizer/crash/fault-injection execution remains locked.

## Verified set
`BUG-ACCE-001..007`

Newly closed:
- `BUG-ACCE-006`: server allows a new absorption to overwrite a sash whose socket0 is already populated; official client requires socket0 == 0, so reversal item policy can be bypassed.
- `BUG-ACCE-007`: `CanWarp()` omits `W_ACCE` and `WarpSet()` does not call `AcceClose()`; stale Acce booleans can keep `CanHandleItem()` blocked after warp.

## Closed without promotion
- reversal leaves element/random/set metadata, but active Acce point application is gated by socket0 and sash is not counted by character set-bonus slots;
- `EFFECT_ACCE_BACK` is initially attached from server `SE_ACCE_BACK` special-effect packet for high-drain sashes; the SetAcce condition alone is not a missing-effect bug.

## Deferred tests
`ACCE-T01..ACCE-T07`. None executed.

## Exact next work
1. combine refine-chain/output-state preservation;
2. sash proto/data + scale/model dependency audit;
3. Acce mutual exclusion against other server windows;
4. remaining lifetime/persistence/reset edges;
5. only then static closure/readiness.

The 22 previously completed subsystems remain unchanged. Acce is not yet counted complete.
Global runtime order remains locked with `DUNGEON-T10` first.

GitHub state is canonical.
