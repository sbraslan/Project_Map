# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Biolog runtime-readiness documentation is now consolidated.
- BIO-T01..T07 cover BUG-BIO-001..007.
- BIO-T08 is retained as the live/external biolog DB proto audit.
- BIO-T09 covers OBS-BIO-003.
- BIO-T10 and BIO-T11 were added for OBS-BIO-001/002 without promoting those observations to verified bugs.
- BIO-T01 is the primary legitimate normal-path Biolog candidate.
- DUNGEON-T10 remains the first future live gate for the whole runtime phase.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Inventory / Item / Special Inventory runtime readiness from `bugs/inventory_items.md` + `tests/inventory_items.md` without executing tests.
2. Preserve DUNGEON-T10 as the first future live runtime gate.
3. Keep the accumulated normal-path candidates queued behind the Dungeon gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
