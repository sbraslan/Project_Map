# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Acce / Sash Static Mapping
**Status:** STATIC MAPPING IN PROGRESS / 4 VERIFIED STATIC BUGS / EXECUTION LOCKED
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

## Next
Continue Acce static mapping:
1. absorbed-stat math and special apply behavior;
2. socket/attribute persistence and reversal/reset;
3. proto/data and client visual dependencies;
4. open-window/warp/item-mutation lifecycle;
5. combine grade/refine-chain and output-placement edge cases.

Do not execute ACCE-T01..T04.
The prepared first future runtime gate remains `DUNGEON-T10`.

GitHub state is canonical.
