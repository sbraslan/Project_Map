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



## Runtime-readiness consolidation — Dungeon Info — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

The Dungeon Info cluster is now normalized into two execution classes without changing any source or game data:

- **Normal-path chain:** DUNGEON-T10 -> DUNGEON-T09 -> DUNGEON-T11 -> DUNGEON-T12.
- **Isolated ASan/debug chain:** DUNGEON-T01 through DUNGEON-T08.

Bug/test mapping:
- T01 -> BUG-DUNGEON-001
- T02 -> BUG-DUNGEON-002
- T03 -> BUG-DUNGEON-003
- T04 -> BUG-DUNGEON-004
- T05 -> BUG-DUNGEON-005
- T06 -> BUG-DUNGEON-006
- T07 -> BUG-DUNGEON-007
- T08 -> BUG-DUNGEON-008
- T09 -> BUG-DUNGEON-009
- T10 -> BUG-DUNGEON-010
- T11 -> BUG-DUNGEON-011
- T12 -> BUG-DUNGEON-012

The first future live gate remains **DUNGEON-T10** because it is reproducible with the current checked-in 9-dungeon configuration and no crafted packet/config change. T09/T11/T12 remain live-pending behind T10. T01-T08 remain isolated/debug-only and must not be executed during the current detection-only phase.

**Next documentation cluster:** Ticket runtime readiness.


## Runtime-readiness consolidation — Ticket — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

Ticket now has a one-to-one bug/test map:
- TICKET-T01 -> BUG-TICKET-001
- TICKET-T02 -> BUG-TICKET-002
- TICKET-T03 -> BUG-TICKET-003
- TICKET-T04 -> BUG-TICKET-004
- TICKET-T05 -> BUG-TICKET-005
- TICKET-T06 -> BUG-TICKET-006
- TICKET-T07 -> BUG-TICKET-007

Execution classes:
- **Normal-path UI validation:** TICKET-T07. This can be observed with legitimate ticket creation/UI behavior and no crafted packet.
- **Isolated/adversarial validation:** TICKET-T01 through TICKET-T06. These require one or more of: modified client input, foreign ticket IDs, disposable DB data, controlled collision setup, invalid admin mode, ASan/UBSan, or crafted non-NUL packets.

Overall runtime order remains unchanged: DUNGEON-T10 is still the first live gate. Ticket's first future normal-path test is **TICKET-T07** after the Dungeon normal-path cluster is cleared.

**Next documentation cluster:** Hunting runtime readiness.


## Runtime-readiness consolidation — Hunting — 2026-09-28
**Documentation status:** READY  
**Execution status:** LOCKED / NOT RUN

Verified bug/test coverage:
- HUNT-T01 -> BUG-HUNT-001
- HUNT-T02 -> BUG-HUNT-002
- HUNT-T03 -> BUG-HUNT-003
- HUNT-T04 -> BUG-HUNT-004
- HUNT-T05 -> BUG-HUNT-005 (broad crash-boundary exploration)
- HUNT-T06 -> BUG-HUNT-005 (direct item-persisted / quest-flags-stale reproduction)

Robustness-only validations:
- HUNT-T07 -> CreateItem nullptr handling; no verified bug ID in the current data snapshot.
- HUNT-T08 -> ignored AddToGround failure; no verified bug ID until runtime/fault-injection reachability is proven.

Execution classes:
- **Legitimate/controlled normal-path:** HUNT-T04 is the primary Hunting live candidate because mission 90 can be completed through the intended progression path. HUNT-T03 is a controlled-state reward-loss check requiring a near-cap gold setup.
- **Modified-client / isolated:** HUNT-T01 and HUNT-T02.
- **Crash consistency / persistence:** HUNT-T05 and HUNT-T06.
- **Fault-injection robustness:** HUNT-T07 and HUNT-T08.

The global first live gate remains **DUNGEON-T10**. Hunting does not preempt that order.

**Next documentation cluster:** Battle Pass runtime readiness.


## Runtime-readiness consolidation — Battle Pass — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

The existing BP-T01..BP-T14 matrix has been classified against the verified Battle Pass bug registry.

Summary:
- several bugs have more than one complementary validation;
- one test remains a broad lifecycle/regression check rather than a unique one-to-one mapping;
- BP-T02 is retained as the primary normal-path observational candidate;
- crash-consistency, leak/instrumentation, and controlled-state cases remain deferred;
- no Battle Pass runtime test has been executed.

The global first live gate remains **DUNGEON-T10**.

**Next documentation cluster:** Achievement runtime readiness.


## Runtime-readiness consolidation — Achievement — 2026-09-28
**Documentation status:** READY
**Execution status:** LOCKED / NOT RUN

Achievement now has deferred runtime coverage for all verified BUG-ACH-001..008. Existing ACH-T01..T08 covered all except BUG-ACH-002, so ACH-T09 was added for the non-transactional cache-rebuild persistence case.

Execution classes:
- **Normal-path gameplay candidates:** ACH-T03 and ACH-T04.
- **Modified-client / isolated interaction:** ACH-T01.
- **Crash consistency / persistence:** ACH-T02 and ACH-T07.
- **Privileged/trusted force paths:** ACH-T05 and ACH-T06.
- **Config-evolution / sanitizer:** ACH-T08.
- **DB atomicity / disposable-data fault test:** ACH-T09.

The global first live gate remains **DUNGEON-T10**.

**Next documentation cluster:** Biolog runtime readiness.

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
