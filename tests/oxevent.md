# OX Event — Deferred Runtime Tests

**Status:** DOCUMENTED / EXECUTION LOCKED / NOT RUN  
**Global first future live gate:** DUNGEON-T09

## OX-T01 — automatic renewal timer collision
After runtime is explicitly unlocked:
1. start OX through the Event Manager path with more than five attendees;
2. timestamp the outer `ox_event_process` callbacks and inner `oxevent_timer` callbacks;
3. run at least three questions;
4. record `static flag`, `m_timedEvent`, `CheckAnswer`, miss-set cleanup and status transitions.

Expected bug signature:
- first question checks at +20s;
- at +35s outer scheduler runs before inner stage 2 and cancels it;
- next question's first timer callback enters stale stage 2 without `CheckAnswer`;
- alternating/corrupted evaluation cadence follows.

Covers `BUG-OX-001`.

## OX-T02 — concurrent final-slot admission
After runtime is explicitly unlocked:
1. configure a small OX player max;
2. bring `ox_map_login_counter` to max-1;
3. have two clients pass the entry dialog nearly simultaneously;
4. trace source-side checks, target-side EnterAttender calls and GD/DG event-flag packets;
5. compare actual `GetAttenderCount()` with the global counter/max.

Expected bug signature: both clients become attendees although one slot remained; counter may undercount or reach max+1.

Covers `BUG-OX-002`.

## OX-T03 — registration-close in-flight warp
After runtime is explicitly unlocked:
1. begin participant entry during the final OPEN window;
2. ensure the source quest passes the OPEN check;
3. let Event Manager switch to CLOSE before target-map login completes;
4. trace `COXEventManager::Enter` on map 113.

Expected bug signature: target state is CLOSE/QUIZ but the exact participant spawn still calls `EnterAttender`.

Covers `BUG-OX-003`.

Do not run these tests while execution lock is active.
