# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Acce / Sash Static Mapping
**Status:** STATIC MAPPING IN PROGRESS / 6 VERIFIED STATIC BUGS / EXECUTION LOCKED
**Machine state:** `STATE.json`
**Active subsystem:** Acce / Sash
**Last completed subsystem:** Costume / Appearance / ChangeLook
**Completed/readiness-covered subsystems:** 22
**Item workflow:** `ITEM_CREATION_AUTOMATION.md`
**First future live gate:** `DUNGEON-T10`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime, crafted-packet, crash, sanitizer and fault-injection execution remains locked until an explicit phase change.

## Active Acce / Sash mapping

New canonical files:
- `systems/acce.md`
- `bugs/acce.md`
- `tests/acce.md`

Verified static findings:
- `BUG-ACCE-001` — final Acce transaction is not bound to an active server-side Acce window/mode.
- `BUG-ACCE-002` — wrong ITEM_COSTUME subtypes can pass the server sash type/subtype predicate because it uses `&&`.
- `BUG-ACCE-003` — absorb material validation accepts every ITEM_ARMOR subtype; `ARMOR_BODY` is incorrectly compared as an item type.
- `BUG-ACCE-004` — combine allows the same inventory cell as both inputs; failure can consume the primary sash and success reaches a stale-pointer/double-remove path.
- `BUG-ACCE-005` — reversal clears attributes after the only target update packet, leaving stale absorbed attributes in client item data/tooltips.
- `BUG-ACCE-006` — absorb accepts an already-occupied sash server-side, overwriting previous absorbed state and consuming the new material.

Key architecture:
- Acce check-in/check-out is client-local bookkeeping.
- Only final accept sends `HEADER_CG_ACCE_REFINE_REQUEST`.
- Server therefore owns full validation responsibility for mode, cells, type/subtype and transaction invariants.

## Global state
The 22 previously completed subsystems remain STATIC COMPLETE with runtime-readiness ownership, plus folded Guild lifecycle coverage.

Acce / Sash is the next deliberately opened subsystem and is **not yet counted as STATIC COMPLETE**.

No runtime test has been executed.

## Item workflow
When the user says `item ekleyeceğiz`, use `ITEM_CREATION_AUTOMATION.md`:
reference VNUM -> compatible template -> unused VNUM/ShapeIndex proposal -> item_proto + companion item/MSM records.

## Newly closed in this pass
- Client visual boundary is now closed: Acce VNUM -> item_list model -> CItemData -> PART_ACCE -> Bip01 Spine2.
- Current 850xx/860xx sash assets use direct WING/item_list GR2 mappings.
- CanWarp/IsHack omit W_ACCE; normal client distance-close mitigates this, so it remains a lifecycle integration note under BUG-ACCE-001 rather than a new destructive bug.

## Next
Continue Acce static mapping:
1. close combine grade/refine-chain and output-placement edge cases;
2. validate extended element/random/set reset semantics;
3. close remaining Acce proto/runtime ownership notes;
4. decide STATIC COMPLETE promotion after the final static pass.

Do not execute ACCE-T01..T06.
The prepared first future runtime gate remains `DUNGEON-T10`.

GitHub state is canonical.
