# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Safebox / Mall runtime-readiness is consolidated.
- New canonical IDs SFB-T01..SFB-T12 separate active-build bugs, observations and dormant SAFEBOX_MONEY bugs.
- Active verified set remains BUG-SAFEBOX-003..005.
- BUG-SAFEBOX-001/002 remain dormant because ENABLE_SAFEBOX_MONEY is OFF.
- Historical OBS-SAFEBOX-001 is subsumed by BUG-SAFEBOX-005.
- No clean normal-player verified-bug candidate exists in the active Safebox set; SFB-T09 is observation-only.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Mailbox runtime readiness from `bugs/mailbox.md` + `tests/mailbox.md`.
2. Preserve active/dormant/observation status exactly.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
