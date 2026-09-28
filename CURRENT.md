# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Runtime-Readiness Consolidation
**Status:** STATIC MAPPING COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Shop / Premium Private Shop runtime-readiness is consolidated.
- New canonical IDs SHP-T01..SHP-T14 cover BUG-SHOP-001..010 and OBS-SHOP-001..004.
- Early provisional duplicate BUG-SHOP-001/002 headings are explicitly superseded by the later canonical Shop index.
- Old stash-cap clipping is preserved as OBS-SHOP-001, not promoted back to a verified normal-flow bug.
- SHP-T02 and SHP-T03 are the primary normal-path Shop candidates.
- DUNGEON-T10 remains the first future live gate.
- No runtime test was executed and no source/game file was changed.

## Exact next work
1. Consolidate Safebox / Mall runtime readiness from `bugs/safebox_mall.md` + `tests/safebox_mall.md`.
2. Preserve canonical-vs-provisional status exactly and add unique test IDs only when needed.
3. Preserve DUNGEON-T10 as the first future live runtime gate.
4. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
