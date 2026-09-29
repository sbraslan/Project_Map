# Monarch — Deferred Runtime Tests

**Status:** DOCUMENTED / EXECUTION LOCKED / NOT RUN  
**Global first future live gate:** DUNGEON-T09

## MON-T01 — election finalizer
After runtime is explicitly unlocked in an isolated DB:
1. create at least two candidacies and deterministic vote rows;
2. invoke registered `elect`;
3. inspect vote-count indices, selected candidate and monarch table/state;
4. inspect game-core monarch update packets.

Expected bug signature: no winner is selected/persisted/published; count path uses voter PID and uninitialized counters.

Covers `BUG-MON-001`.

## MON-T02 — setmonarch live synchronization
After runtime is explicitly unlocked:
1. record current monarch state on two game cores;
2. invoke `setmonarch <valid player>`;
3. verify DB row;
4. trace DG packet header and both game-core `CMonarch` states.

Expected bug signature: DB changes while running game-core state remains old because no UPDATE_MONARCH_INFO is handled.

Covers `BUG-MON-002`.

## MON-T03 — rmmonarch DELETE result/state divergence
After runtime is explicitly unlocked:
1. start with a valid monarch;
2. invoke `rmmonarch <name>`;
3. trace both consecutive `DelMonarch` calls and `iRet`;
4. inspect persistent row and live game-core state.

Expected bug signature: SQL DELETE affects the row, uiNumRows remains zero, DelMonarch returns false without clearing runtime state, and no authoritative live-state refresh is broadcast.

Covers `BUG-MON-003`.

## MON-T04 — invalid mtax boundary
After runtime is explicitly unlocked:
1. use the actual monarch character;
2. issue `mtax 0`, `mtax 51` and a larger isolated test value;
3. read `trade_tax` event flag after each command;
4. observe notices/cooldown.

Expected bug signature: invalid value is reported but still becomes the event flag and consumes cooldown.

Covers `BUG-MON-004`.

Do not run these tests while the execution lock is active.


## MON-T05 — rapid cross-command treasury boundary race
After runtime is explicitly unlocked in an isolated environment:
1. set treasury to a value that can fund either the chosen pair individually but not both in sequence (for example around the `mtr` + `mmob` boundary);
2. ensure MI_TRANSFER and MI_SUMMON are both ready;
3. issue valid `mtr` and `mmob` requests before the first DG treasury update returns;
4. trace local prechecks, gameplay effects, DB `DecMoney` returns, and DG DEC fanout.

Expected bug signature: both effects occur while DB accepts only one deduction; rejected deduction has no effect rollback.

Covers `BUG-MON-005`.

## MON-T06 — tax cooldown enforcement
After runtime is explicitly unlocked:
1. issue a valid `mtax` change as monarch;
2. immediately issue another valid `mtax` change;
3. inspect MI_TAX timestamps and resulting `trade_tax` flag.

Expected bug signature: second tax change succeeds during the configured MI_TAX cooldown.

Covers `BUG-MON-006`.

## MON-T07 — remote transfer target disappears
After runtime is explicitly unlocked on a multi-core setup:
1. choose a same-empire remote target visible in P2P CCI;
2. invoke `mtr <target>`;
3. make the target disconnect/change ownership after source validation but before receiver `FindPC`;
4. inspect receiver warp, treasury and MI_TRANSFER cooldown.

Expected bug signature: receiver performs no warp, but treasury deduction/cooldown was already committed from source.

Covers `BUG-MON-007`.

Do not run these tests while the global execution lock is active.
