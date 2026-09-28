# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Exchange / Trade runtime-readiness is consolidated.
- New canonical test IDs EXC-T01..EXC-T10 replace ambiguous legacy EX-Txx / EXCHANGE-Txx references for future continuation.
- BUG-EXCHANGE-001..003 have clear test coverage.
- Historical BUG-EXCHANGE-004 is reused for two distinct verified findings; the collision is documented without renumbering.
- Observation numbering around packet-init vs AddGold/Cheque logic is also historically inconsistent and is now referenced descriptively.
- EXC-T01 and EXC-T02 are the primary normal-path Exchange candidates.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Shop / Premium Private Shop runtime readiness from `bugs/shop.md` + `tests/shop.md`.
2. Preserve historical IDs but use canonical subsystem test IDs when legacy identifiers collide.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
