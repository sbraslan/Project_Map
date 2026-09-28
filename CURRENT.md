# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** First Runtime Gate Ready
**Status:** PREFLIGHT COMPLETE / RUNTIME EXECUTION LOCKED
**Machine state:** `STATE.json`
**Global audit:** `READINESS_AUDIT.md`
**First gate:** `DUNGEON_T10_HANDOFF.md`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime, crafted-packet, sanitizer, crash-consistency and fault-injection execution stay locked until the user explicitly changes phase.

## Just closed
- DUNGEON-T10 preflight/handoff documentation is COMPLETE.
- The gate is normal-flow only: login -> minimap Dungeon Info button -> observe window.
- Required evidence, result classes and stop conditions are fully defined in `DUNGEON_T10_HANDOFF.md`.
- Canonical bug is `DINFO::BUG-DUNGEON-010`.
- Static prediction remains: tracked 9-entry config, window opens, dungeon rows are missing.
- T09/T11/T12 remain blocked until T10 has a recorded runtime result.
- FIX-DUNGEON-010 remains reference-only and must not be applied before evidence capture.
- No runtime test was executed and no source/game file was changed.

## Exact next work
Current documentation-only work for the first gate is exhausted.

The next actual project action is **DUNGEON-T10 live execution**, but it remains locked under the current Detection / Mapping Only phase. It may begin only after the user explicitly changes phase/authorizes runtime execution.

Until then:
1. do not run DUNGEON-T10;
2. do not apply any Dungeon Info fix;
3. keep source/game repositories immutable;
4. use `STATE.json` + `DUNGEON_T10_HANDOFF.md` as the resume point.

GitHub state is canonical.
