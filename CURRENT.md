# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Achievement runtime-readiness documentation is now consolidated.
- ACH-T01..T08 were mapped to the existing verified Achievement bugs.
- A coverage gap was found for BUG-ACH-002; new deferred test ACH-T09 now covers the cache-rebuild atomicity case.
- ACH-T03 and ACH-T04 are the primary legitimate normal-path Achievement candidates.
- DUNGEON-T10 remains the first future live gate for the whole runtime phase.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Biolog runtime readiness from `bugs/biolog.md` + `tests/biolog.md` without executing tests.
2. Preserve DUNGEON-T10 as the first future live runtime gate.
3. Keep Ticket T07, Hunting T04, Battle Pass T02, and Achievement T03/T04 queued behind the Dungeon normal-path cluster.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
