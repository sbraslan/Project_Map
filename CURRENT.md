# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Dungeon Info runtime-readiness documentation is now consolidated.
- DUNGEON-T01..T12 are mapped one-to-one to BUG-DUNGEON-001..012.
- Isolated/debug set: T01-T08.
- Normal-path chain: T10 -> T09 -> T11 -> T12.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate the Ticket runtime-readiness cluster from `bugs/ticket.md` + `tests/ticket.md` without executing tests.
2. Then consolidate Hunting readiness.
3. Preserve DUNGEON-T10 as the first future live runtime gate unless the user explicitly changes the order.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
