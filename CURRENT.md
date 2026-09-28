# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Party Match runtime-readiness is consolidated.
- `tests/party_match.md` did not previously exist; canonical PMATCH-T01..PMATCH-T05 deferred tests were created.
- PMATCH-T01..T03 cover BUG-PMATCH-001..003 one-to-one.
- PMATCH-T04/T05 are explicitly unpromoted robustness tests for duplicate SEARCH/HOLD and WarpSet failure ordering.
- PMATCH-T03 is the primary normal-path Party Match candidate.
- Party and Party Match remain separate ownership domains.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Dungeon Core runtime readiness from `bugs/dungeon_core.md` + `tests/dungeon_core.md`.
2. Keep Dungeon Core distinct from the already-consolidated Dungeon Info subsystem.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
