# Arena / PvP Duel — Deferred Runtime Tests

**Status:** DOCUMENTED / EXECUTION LOCKED / NOT RUN  
**Global first future live gate:** DUNGEON-T09

## ARENA-T01 — normal opponent eligibility
After runtime is explicitly unlocked:
1. use two eligible level-qualified players near NPC 20017;
2. ensure neither is an Arena member/observer;
3. enter opponent name through the deployed arena quest;
4. trace `arena.is_in_arena(opp_vid)`.

Expected bug signature: idle opponent returns 0 and the quest rejects before confirmation/start_duel.

Covers `BUG-ARENA-001`.

## Deferred-after-fix probes
Do not execute these as canonical bug tests until BUG-ARENA-001/start reachability is repaired:
- map112 routing / WarpSet result and arena-slot reservation;
- duel timeout client reset packet symmetry A vs B;
- observer Arena pointer/mode teardown across same-process return.

No Arena runtime test may run while the execution lock is active.
