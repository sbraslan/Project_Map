# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- World Lottery runtime-readiness is consolidated.
- WLOT-T01..T14 cover BUG-WLOT-001..014; WLOT-T15 adds complementary BUG-WLOT-003 login/status-width validation.
- WLOT-T04 and WLOT-T10 are the primary ordinary/current-flow candidates.
- Ranking-endpoint tests WLOT-T11/T12 remain dormant; WLOT-T13/T14 remain crash/fault-injection only.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate World Boss runtime readiness from `bugs/world_boss.md` + `tests/world_boss.md`.
2. Preserve server/client/session/persistence ownership without inventing duplicate bug IDs.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
