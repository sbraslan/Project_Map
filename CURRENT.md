# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Dungeon Core runtime-readiness is consolidated.
- DCORE-T01..T04 cover BUG-DUNGEON-001..004 one-to-one.
- DCORE-T01/DCORE-T02 are the primary legitimate Dungeon Core candidates.
- DCORE-T03 is ASan/debug lifetime validation; DCORE-T04 is controlled script/regression coverage.
- Dungeon Core remains separate from Dungeon Info.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Battle Field runtime readiness from `bugs/battle_field.md` + `tests/battle_field.md`.
2. Preserve cross-system Party/Ranking ownership; do not duplicate their bug IDs into Battle Field.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
