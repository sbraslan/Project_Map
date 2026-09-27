# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Battle Pass runtime-readiness classification is complete.
- BP-T01..T14 are categorized against the existing verified Battle Pass bug registry.
- Some verified bugs have multiple complementary tests; one lifecycle regression test remains broad rather than uniquely mapped.
- BP-T02 is retained as the primary normal-path observational Battle Pass candidate.
- DUNGEON-T10 remains the first future live gate for the whole runtime phase.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Achievement runtime readiness from `bugs/achievement.md` + `tests/achievement.md` without executing tests.
2. Preserve DUNGEON-T10 as the first future live runtime gate.
3. Keep Ticket T07, Hunting T04 and Battle Pass T02 queued behind the Dungeon normal-path cluster.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
