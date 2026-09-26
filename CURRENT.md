# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Status:** ACTIVE
**Machine state:** `STATE.json`
**Last updated:** 2026-09-26

## Hard rule
Only `sbraslan/Project_Map` is writable.

The following repositories are strictly read-only:
- `Project_ClientSrc`
- `Project_ServerSRC`
- `Project_Binary`
- `Project_Game`
- `Project_DumpProto`

No C++, Python, quest, config, game-data, source, binary or gameplay file may be edited, committed or pushed.

## What we do now
- inspect code;
- map architecture and data flow;
- identify bugs, security/correctness risks and edge cases;
- record evidence in `systems/`, `bugs/`, `tests/`, `CURRENT.md` and `STATE.json`;
- use GitHub history as the durable project memory.

Runtime/fault-injection tests may be documented as future tests, but they are **not executed as part of the current phase** unless the user explicitly changes the phase later.

## Static mapping checkpoint
- Hunting System -> STATIC COMPLETE, BUG-HUNT-001..005.
- Ticket System -> STATIC COMPLETE, BUG-TICKET-001..007.
- Dungeon Info -> STATIC COMPLETE, BUG-DUNGEON-001..012.
- Other subsystem statuses remain in `INDEX.md`.

## Current direction
Continue read-only source inspection and improve/extend the canonical maps and bug registries.

The previously prepared Dungeon remediation notes are reference-only. They do not authorize source changes.

## Startup
For a normal "ilerleyelim" turn:
1. read `STATE.json`;
2. read `CURRENT.md`;
3. read `WORKFLOW.md`;
4. inspect only the exact source symbols needed;
5. write findings only to `Project_Map`.

Do not reconstruct state from old chats. GitHub state is canonical.
