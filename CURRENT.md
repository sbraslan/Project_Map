# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Inventory / Item / Special Inventory runtime-readiness documentation is consolidated.
- Verified bug coverage is recorded for BUG-ITEM-001..004 and BUG-ITEM-006..008.
- No canonical BUG-ITEM-005 exists; the numbering gap is preserved.
- ITEM-T05/T07 remain observation/regression validation for OBS-ITEM-001/002.
- ITEM-T04/T08 are regression checks; ITEM-T13 is a dataset-audit dependency.
- Mixed legacy SWITCHBOT tests were not folded into Inventory ownership because Switchbot has its own canonical subsystem.
- ITEM-T01 is the primary Inventory normal-path candidate.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Switchbot runtime readiness from `bugs/switchbot.md` + `tests/switchbot.md`.
2. Resolve canonical Switchbot test IDs there rather than relying on duplicate legacy IDs in `tests/inventory_items.md`.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
