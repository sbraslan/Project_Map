# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Hunting runtime-readiness documentation is now consolidated.
- HUNT-T01..T04 map to BUG-HUNT-001..004.
- HUNT-T05 and HUNT-T06 both validate BUG-HUNT-005 from different crash-consistency boundaries.
- HUNT-T07/T08 remain robustness-only checks with no promoted bug ID.
- HUNT-T04 is the primary legitimate Hunting live candidate.
- DUNGEON-T10 remains the first future live gate for the whole runtime phase.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Battle Pass runtime readiness from `bugs/battle_pass.md` + `tests/battle_pass.md` without executing tests.
2. Preserve DUNGEON-T10 as the first future live runtime gate.
3. Keep Ticket T07 and Hunting T04 queued behind the Dungeon normal-path cluster.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
