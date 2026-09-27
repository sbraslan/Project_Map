# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Ticket runtime-readiness documentation is now consolidated.
- TICKET-T01..T07 map one-to-one to BUG-TICKET-001..007.
- TICKET-T07 is the normal-path UI validation target.
- TICKET-T01..T06 remain isolated/adversarial tests.
- DUNGEON-T10 remains the first future live gate for the whole runtime phase.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Hunting runtime readiness from `bugs/hunting.md` + `tests/hunting.md` without executing tests.
2. Preserve DUNGEON-T10 as the first future live runtime gate.
3. Keep Ticket T07 queued as the first Ticket normal-path validation.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
