# CURRENT — Canonical Active Checkpoint

**Active subsystem:** Dungeon Info
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Last updated:** 2026-09-26

## Startup read set
For a normal "ilerleyelim" turn read only:
1. `STATE.json`
2. `CURRENT.md`
3. `systems/dungeon_info.md`

Conditional:
- `bugs/dungeon_info.md` only when validating/recording a bug.
- `tests/dungeon_info.md` only for runtime/fault-injection work.
- source repos: search exact symbols first, then fetch exact files/ranges.

Do **not** reconstruct state from old chats. GitHub state is canonical.

## Just closed
Ticket System -> **STATIC COMPLETE**.
Verified Ticket bugs: `BUG-TICKET-001..007`.

New close findings:
- non-NUL fixed-char CG fields can drive server OOB string reads (006);
- normal-user log contract is server 40 / C++ client cache 10 / UI 20x10 pages (007);
- Ticket packet sequence/framing and C++ server-command fallback were otherwise consistent.

## Current Dungeon Info state
Already mapped:
- `game/src/DungeonInfo.cpp`
- `input_main.cpp::DungeonInfo`
- `game/src/packet.h`
- `UserInterface/PythonDungeonInfo.cpp/.h`
- `PythonNetworkStreamPhaseGame.cpp`

Verified Dungeon bugs: `BUG-DUNGEON-001..006`.

## Exact next work
1. Validate CG/GC packet-info size/sequence and dungeon index boundaries.
2. Audit config parser invariants and fixed packet-array capacities.
3. Audit ranking DB/result lifecycle and Python binding boundaries.
4. Close client clear/reload/state lifecycle.
5. Decide Dungeon Info STATIC COMPLETE and update tests/checkpoint.

## End-of-turn write rule
When meaningful progress is made:
1. update `systems/dungeon_info.md`;
2. update `bugs/dungeon_info.md` / `tests/dungeon_info.md` only if affected;
3. overwrite this file;
4. update `STATE.json`.

Keep this file short and overwrite-only.
