# RUNTIME — In-Game / Fault-Injection Validation

**Status:** ACTIVE
**Started:** 2026-09-26

Static mapping remains canonical in `INDEX.md` and per-system files. This file is the short cursor for the next phase; do not bulk-load every bug registry into one chat.

## Goal
Validate confirmed static bugs against the running GAME/client, distinguish immediately reproducible normal-path failures from crafted-input/fault-injection issues, then fix them subsystem-by-subsystem.

## Read discipline
For each runtime turn:
1. read `STATE.json`, `CURRENT.md`, `RUNTIME.md`;
2. select one subsystem/test cluster;
3. read only that subsystem's `bugs/<name>.md` and `tests/<name>.md`;
4. fetch only the exact source symbols needed;
5. record result before moving on.

## Initial validation order
### Stage A — normal-path, no crafted packets
Start with defects that should reproduce using current checked-in data/UI:
- Dungeon: T10 UI list creation, T09 ranking SQL, T11 GLOBAL flag parsing, T12 cooldown wrap.
- Ticket: normal-user pagination mismatch (T07 in Ticket test plan).
- Hunting: legitimate level-90 terminal boundary and reward persistence behavior where safe.

### Stage B — isolated ASan/debug or modified client
- Dungeon: index/slot/config overflow tests T01-T08.
- Ticket: foreign ID, boundary, non-NUL packet tests.
- Hunting: invalid type/action and CreateItem/ground fault injection.

### Stage C — crash consistency / persistence
- Hunting item-vs-quest reward commit window.
- other subsystem persistence tests already recorded in their individual test files.

## Current next target
**Dungeon Info normal-path cluster: DUNGEON-T10 -> T09 -> T11 -> T12.**

These four tests require no intentionally corrupt config/packet for the first pass and directly exercise the current snapshot.
