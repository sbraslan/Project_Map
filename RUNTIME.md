# RUNTIME — In-Game / Fault-Injection Validation

**Status:** DEFERRED — static mapping complete; runtime execution still locked
**Started:** 2026-09-26

Static mapping is now complete across the subsystem index. This file is the short cursor for the next phase; do not bulk-load every bug registry into one chat.

## Goal
Future-only test inventory. During the current detection/mapping phase, do not execute these tests and do not modify source/game files. Keep this file only as deferred validation notes.

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

### Sung Mahi Tower — prepared cluster
Detailed deferred plan: `tests/sung_mahi_tower.md`.

Prepared validations:
- SMT-T01 — missing tower quest runtime normal path (BUG-SMT-001)
- SMT-T02 — missing monthly reset Lua library (BUG-SMT-002)
- SMT-T03 — monthly mailbox memcpy / unterminated-string ASan check (BUG-SMT-003)
- SMT-T04 — cross-year same-month rollover simulation (BUG-SMT-004)
- SMT-T05 — dark king 7591 resistance comparison (BUG-SMT-005)
- SMT-T06 — tower-only item proto availability (BUG-SMT-006)

Do not run this cluster until runtime execution is explicitly enabled.

## Current next target
**DUNGEON-T10 live reproduction is the gate.**

Preflight result:
- current server config has 9 dungeon entries;
- client receives/stores them before GC OPEN;
- UI `Initialize()` creates list rows only in the zero-count branch;
- therefore T10 has a deterministic normal-path trigger.

Dependency:
`T10 live repro -> fix/bypass T10 -> T09 -> T11 -> T12`.

T09, T11 and T12 are code-path confirmed but remain **LIVE PENDING** because the broken T10 list prevents normal dungeon selection/UI inspection.

### Exact live action
On the current running client/server:
1. log in normally;
2. click the Dungeon Info icon next to the minimap;
3. capture whether the window has dungeon rows.

Reproduction criterion for BUG-DUNGEON-010:
**window opens, but the dungeon list is empty/missing even though the server config contains 9 dungeons.**

No source/config modification is needed for this first test.


## Patch readiness
First-cluster fix plan is ready at `fixes/dungeon_info.md`.

Prepared without modifying source repos:
- FIX-DUNGEON-010: correct UI list-construction control flow.
- FIX-DUNGEON-009: repair missing whitespace before ranking LEFT JOIN.
- FIX-DUNGEON-011: accept documented numeric `1 = GLOBAL` while retaining literal `GLOBAL` compatibility.
- FIX-DUNGEON-012A: use signed/clamped cooldown arithmetic to eliminate uint32 wrap.
- FIX-DUNGEON-012B: intentionally deferred data-semantics decision; current quest flags represent different timer meanings, so no guessed COOLDOWN values will be written.

Important: Flame/Snow `exit_time` and Dragon `dragon_lair_time` are timestamp-style flags, but they do not all represent the same gameplay window. Arithmetic safety can be fixed generically; displayed cooldown policy must be chosen per dungeon.


## Current phase lock
This file is **not the active execution cursor**.

Current user rule:
- inspect only;
- detect/map only;
- write findings only to `Project_Map`;
- no source, Python, C++, quest, config or game-data changes;
- no runtime/fault-injection execution until the user explicitly changes phase.
