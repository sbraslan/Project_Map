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

## MON-T03 — rmmonarch double deletion
After runtime is explicitly unlocked:
1. start with a valid monarch;
2. invoke `rmmonarch <name>`;
3. trace both consecutive `DelMonarch` calls and `iRet`;
4. inspect persistent row and live game-core state.

Expected bug signature: first mutation can remove state, second lookup returns failure, and no authoritative live-state refresh is broadcast.

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
