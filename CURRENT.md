# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Guild Storage runtime-readiness documentation is consolidated.
- Verified coverage is recorded for BUG-GS-003, BUG-GS-004 and BUG-GS-007..011.
- Legacy candidate GS-003/004 are treated as superseded by their verified records.
- BUG-CANDIDATE-GS-001/002/005/006 remain candidates; no verified IDs were invented.
- GS-T01..T06 and GS-T09 remain baseline/regression coverage.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Exchange / Trade runtime readiness from `bugs/exchange.md` + `tests/exchange.md`.
2. Preserve verified-vs-candidate status exactly; do not promote findings without evidence.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
