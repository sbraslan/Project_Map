# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Switchbot runtime-readiness is consolidated.
- Duplicate legacy SWITCHBOT-T01..T06 identifiers were preserved but marked non-canonical.
- New unique canonical IDs SWB-T01..SWB-T07 now map BUG-SWITCHBOT-001..005 and OBS-SWITCHBOT-001/002.
- SWB-T01 and SWB-T03 are the primary legitimate monitored candidates.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Guild Storage runtime readiness from `bugs/guild_storage.md` + `tests/guild_storage.md`.
2. Use only canonical subsystem test IDs; do not reuse legacy mixed-file duplicates.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
