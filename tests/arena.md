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

## ARENA-T02 — Weekly/BattleArena deployment preflight
After runtime is explicitly unlocked:
1. verify map-location registry for 190/191/192;
2. verify expected battlearena map/regen files;
3. invoke `weeklyevent 1` only in an isolated environment;
4. trace `CBattleArena::Start`, status, event flag and first map lookup.

Expected bug signature: Start reports success/running state while target map has no deployed route/data.

Covers `BUG-ARENA-002`.

## ARENA-T03 — BattleArena force-end timing
After runtime is explicitly unlocked in a repaired isolated BattleArena deployment:
1. start Weekly/BattleArena;
2. enter an active monster-wave phase;
3. issue `weeklyevent` again to invoke ForceEnd;
4. record immediate GM response and `m_bForceEnd`;
5. trace subsequent `battle_arena_event` states/timestamps.

Expected bug signature: GM receives “Weekly Event End”, but replacement state-3 event continues normal monitoring/spawn progression because `m_bForceEnd` is never consumed.

Covers `BUG-ARENA-003`.

## Deferred-after-fix classic probes
Do not execute these as canonical bug tests until BUG-ARENA-001/start reachability is repaired:
- map112 routing / WarpSet result and arena-slot reservation;
- duel timeout client reset packet symmetry A vs B;
- observer Arena pointer/mode teardown across same-process return.

No Arena runtime test may run while the execution lock is active.
