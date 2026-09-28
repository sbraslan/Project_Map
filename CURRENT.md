# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Ranking runtime-readiness is consolidated.
- RANK-T01..T04 cover BUG-RANK-001..004.
- New RANK-T07 covers BUG-RANK-005.
- RANK-T06 is now the deferred validation for BUG-RANK-007.
- BUG-RANK-006 remains RETRACTED / RESERVED and is excluded from the runtime validation matrix.
- RANK-T05 remains conditional coverage for the dormant generic PARTY ranking API gap.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Party runtime readiness from `bugs/party.md` + `tests/party.md`.
2. Preserve unpromoted/dormant integration gaps without inventing bug IDs.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
