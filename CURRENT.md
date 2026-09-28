# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Mailbox runtime-readiness is consolidated.
- MAIL-T01..T15 cover canonical BUG-MAIL-001..012.
- MAIL-T16/T17 cover OBS-MAIL-001/002.
- MAIL-T18 was added for OBS-MAIL-003 without promoting it to a bug.
- MAIL-T07 and MAIL-T11 are the primary ordinary-flow Mailbox candidates.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Ranking runtime readiness from `bugs/ranking.md` + `tests/ranking.md`.
2. Preserve the existing retracted BUG-RANK-006 status.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
