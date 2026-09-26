# CURRENT — Canonical Active Checkpoint

**Active phase:** Runtime / In-Game Validation
**Status:** ACTIVE
**Machine state:** `STATE.json`
**Runtime cursor:** `RUNTIME.md`
**Last updated:** 2026-09-26

## Startup read set
For a normal "ilerleyelim" turn read only:
1. `STATE.json`
2. `CURRENT.md`
3. `RUNTIME.md`

Then read only the selected subsystem bug/test files.
Do **not** reconstruct state from old chats. GitHub state is canonical.

## Static phase checkpoint
- Hunting System -> STATIC COMPLETE, BUG-HUNT-001..005.
- Ticket System -> STATIC COMPLETE, BUG-TICKET-001..007.
- Dungeon Info -> STATIC COMPLETE, BUG-DUNGEON-001..012.
- Current `INDEX.md` has no remaining PARTIAL subsystem.
- Guild lifecycle remains represented as `MAPPED WITH GUILD STORAGE`, not as an open PARTIAL audit.

## First runtime cluster
**Gate:** DUNGEON-T10 live reproduction.
**Patch plan:** `fixes/dungeon_info.md` READY; source repos unchanged.

Preflight confirms the full normal path and the current 9-entry dataset. T10 blocks the other normal UI tests because no dungeon rows are created.

After T10 is reproduced and fixed/bypassed:
1. DUNGEON-T09 — ranking SQL failure.
2. DUNGEON-T11 — numeric GLOBAL flag parse mismatch.
3. DUNGEON-T12 — unset/expired cooldown uint32 wrap.

T09/T11/T12 are code-path confirmed but not yet marked runtime PASS.

Minimal fixes for T10/T09/T11/T12A are prepared. T12B cooldown-data semantics is deliberately deferred because current quest flags encode different gameplay timers.

## Write rule
After each meaningful runtime result:
1. update the selected `tests/<system>.md` with PASS/FAIL/repro evidence;
2. update `bugs/<system>.md` if severity/reachability changes;
3. update `RUNTIME.md` cursor;
4. overwrite this file and update `STATE.json`.
