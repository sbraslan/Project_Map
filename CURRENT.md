# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Dungeon Core
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/dungeon_core.md`
**Last updated:** 2026-09-26

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Closed this turn — Party Match
Party Match is now **STATIC COMPLETE**.

Verified Party Match bugs:
- `BUG-PMATCH-001`
- `BUG-PMATCH-002`
- `BUG-PMATCH-003`

## Active Dungeon Core checkpoint
Initial roots mapped:
- `dungeon.h/.cpp`;
- `questlua_dungeon.cpp`;
- character dungeon association;
- party dungeon association;
- private-map create/destroy;
- dead/exit/jump event lifecycle.

Initial candidates:
- event lookup null-ordering;
- JoinParty state mutation before map validation;
- raw party lifetime in dungeon maps.

## Exact next work
1. Close CHARACTER::SetDungeon membership lifecycle.
2. Close party/dungeon pointer lifetime.
3. Close event callback lifetime/null ordering.
4. Map quest dungeon creation/join flows.
5. Promote only verified bugs.

GitHub state is canonical.
