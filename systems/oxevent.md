# OX Event — Static Map

**Status:** STATIC MAPPING IN PROGRESS / 5 VERIFIED BUGS  
**Phase:** Detection / Mapping Only  
**Opened:** 2026-09-29  
**Source/Game repositories:** READ-ONLY  
**Runtime:** LOCKED / NOT RUN

## Scope
- OX manager state and participant containers;
- deployed entry/GM quest;
- automatic Event Manager orchestration;
- quiz loading, timing, answer evaluation and elimination;
- player-cap enforcement;
- map 113 login/warp lifecycle;
- rewards, close/reset and multi-round behavior.

## Verified source roots
- `Project_ServerSRC/game/src/OXEvent.cpp`
- `Project_ServerSRC/game/src/OXEvent.h`
- `Project_ServerSRC/game/src/questlua_oxevent.cpp`
- `Project_ServerSRC/game/src/cmd_oxevent.cpp`
- `Project_ServerSRC/game/src/event_manager.cpp`
- `Project_ServerSRC/game/src/event_manager.h`
- `Project_ServerSRC/game/src/event.cpp`
- `Project_ServerSRC/game/src/event_queue.cpp`
- `Project_ServerSRC/game/src/stable_priority_queue.h`
- `Project_ServerSRC/game/src/input_login.cpp`
- `Project_ServerSRC/game/src/questlua_game.cpp`
- `Project_ServerSRC/game/src/questmanager.cpp`
- `Project_ServerSRC/db/src/ClientManagerEventFlag.cpp`
- `Project_Game/share/locale/europe/quest/e_event/oxevent.quest`
- `Project_Game/share/locale/europe/oxquiz.lua`
- `Project_Game/share/locale/europe/map/metin2_map_oxevent/*`
- deployed core CONFIG files.

## Deployment / feature gates
- `ENABLE_EVENT_MANAGER` active.
- `ENABLE_OX_RENEWAL` active.
- `OX_REWARD_UPDATE` disabled.
- deployed `quest_list` contains `e_event/oxevent.quest`.
- `oxquiz.lua` contains 346 tracked questions, all registered at quiz level 1.
- OX map is map 113.
- current tracked CONFIG ownership: map 113 exists only on `chan/ch99/core99`.
- Event Manager channel is 99, so the automatic OX scheduler and OX map live on the same channel/core ownership domain.

## Player entry flow
Deployed NPC quest:
`20011 chat`
→ require `oxevent_status == 1 (OPEN)`
→ level range / cooldown / player-limit checks
→ `pc.warp(896500, 24600)`
→ route to map 113 / ch99 core99
→ game login path
→ `COXEventManager::Enter(ch)`
→ exact coordinate test
→ `EnterAttender`
→ PID inserted in `m_map_char` and `m_map_attender`.

Audience entry is exact coordinate `896300,28900`.

## Player limit flow
Quest `check_limit()` blocks only when:
`ox_map_login_counter == ox_map_player_max`.

On map login/enter:
`counter = game.get_event_flag("ox_map_login_counter")`
→ `game.set_event_flag(counter + 1)`
→ `CQuestManager::RequestSetEventFlag`
→ `HEADER_GD_SET_EVENT_FLAG`
→ DB `CClientManager::SetEventFlag`
→ `HEADER_DG_SET_EVENT_FLAG` broadcast
→ local event-flag copy finally updates.

The write is therefore asynchronous and is not an atomic reservation.

### BUG-OX-002 — player cap accounting is non-authoritative and can be exceeded/corrupted
Two players can both pass the source-side limit check while one slot remains. The target-side authoritative `EnterAttender` performs no cap check.

Depending on DB-echo timing, the login counter either loses one increment or reaches max+1. Because `check_limit()` tests equality rather than `>=`, a max+1 counter also reopens future admission.

There is also a deterministic duplicate-login path: a participant who disconnects/relogs while still at the exact attendee spawn is inserted into the same PID map entry again (no attendee growth) but the deployed `when login or enter` quest increments `ox_map_login_counter` again. The logout cooldown is only checked at the NPC entry flow, not on map relog. Repeating this can consume or push the counter beyond the configured cap without adding unique attendees.

Promoted as `BUG-OX-002`.

## Registration close boundary

### BUG-OX-003 — in-flight OPEN admission remains valid after status becomes CLOSE/QUIZ
The source quest validates OPEN before starting the cross-core warp. Target `COXEventManager::Enter` rejects only `OXEVENT_FINISH`; it accepts both CLOSE and QUIZ.

Therefore:
OPEN check on source
→ cross-core warp starts
→ Event Manager closes registration / starts quiz
→ player reaches map 113
→ `Enter` sees CLOSE/QUIZ but still calls `EnterAttender`.

Promoted as `BUG-OX-003`.

## Automatic Event Manager flow
Only channel 99 processes event queues.

`EVENT_TYPE_OX`
→ `CEventManager::SetOXEvent(true)`
→ load `oxquiz.lua`
→ status OPEN
→ create outer `ox_event_process`
→ registration countdown
→ state CLOSE
→ every 35 seconds, if attendees > 5, call `COXEventManager::Quiz(1,30)`.

Current constants:
- waiting time: 5 minutes;
- rounds: 3;
- winner threshold: 5;
- reward: item 50109 x1.

## Inner quiz timer under ENABLE_OX_RENEWAL
`Quiz(1,30)` subtracts 15 and creates inner timer at +15s.

Function-local static `flag` in `oxevent_timer`:
- stage 0 at +15s → flag=1 → +5s;
- stage 1 at +20s → `CheckAnswer` → flag=2 → +15s;
- stage 2 at +35s → `WarpToAudience` → status CLOSE → flag=0 → end.

The outer Event Manager also schedules the next quiz exactly +35s after starting the prior quiz.

The event queue is stable for equal keys. The outer +35 event was enqueued before the inner stage-2 +35 event, so the outer event executes first.

### BUG-OX-001 — 35-second scheduler collision cancels cleanup and corrupts the next quiz stage
At +35:
1. outer `ox_event_process` runs first;
2. it calls `Quiz(1,30)`;
3. `Quiz` sees the previous `m_timedEvent` and cancels it before stage 2 executes;
4. function-local static `flag` remains 2;
5. a new inner timer is created for the new question;
6. that new timer's first callback enters stage 2 immediately, performs no `CheckAnswer`, warps the previous miss set and resets the flag.

Result: automatic renewal OX corrupts the question cadence; alternate questions can skip answer evaluation and previous losers are cleaned up during the following question rather than before it.

Promoted as `BUG-OX-001`.

## Multi-round reset

Automatic Event Manager is explicitly configured for `OX_ROUND_COUNT = 3`.

When a round ends and rounds remain:
`ox_event_process / FINISH`
→ `COXEventManager::CloseEvent()`
→ clear local OX participant containers
→ set outer state OPEN
→ reset registration countdown
→ set OX status OPEN.

The deployed entry quest's `ox_map_login_counter` is not reset by this path. That flag is reset only by the quest helper `cleanup_event()` / GM manual cleanup.

### BUG-OX-004 — automatic round restart keeps the previous round's admission counter
The automatic three-round scheduler resets local OX maps but not the quest's persistent admission counter. As a result, round 2/3 capacity is calculated from cumulative prior-round login count rather than the new round.

If round 1 reached `ox_map_player_max`, the next OPEN round starts with `counter == max` and the deployed NPC blocks all new entrants. If the counter was below max, only the remaining cumulative difference is available.

Promoted as `BUG-OX-004`.

## Manual GM controls versus automatic scheduler
The deployed GM quest exposes `oxevent.end_event()` and `oxevent.end_event_force()`.

`end_event_force`:
→ `COXEventManager::CloseEvent()`
→ status FINISH.

It does **not** call `CEventManager::SetOXEvent(false)` and cannot cancel the outer `m_pOXEvent` process event.

Under active `ENABLE_EVENT_MANAGER`, `Initialize()` invoked by CloseEvent does not clear the loaded quiz vector.

### BUG-OX-005 — deployed force-end can leave the automatic OX scheduler alive
When OX was started by Event Manager, using the deployed GM force-end closes current players/inner timer and marks status FINISH, but the outer scheduler remains queued.

At its next scheduled callback the outer state machine can continue its registration/quiz/round logic and set OX status again. Because quiz data remains loaded in the Event Manager build, this is not merely a stale empty scheduler.

Promoted as `BUG-OX-005`.

## Answer flow
`CheckAnswer(answer)`
→ iterate `m_map_attender`
→ find local character by PID
→ compare current coordinates to O/X answer rectangle
→ correct: success/chat effect, remains attendee
→ incorrect: remove attendee, add PID to `m_map_miss`
→ `WarpToAudience()` moves misses to audience positions with `Show(MAP_OXEVENT,...)`
→ miss map cleared.

## Close flow
`CloseEvent()`
→ cancel inner timer
→ WarpSet all tracked `m_map_char` players to empire starts
→ `Initialize()`
→ clear local participant sets
→ status FINISH.

The OX map has one tracked core owner, so no duplicate-map-manager bug comparable to Marriage map 81 is present.

## Closed / scoped observations
- Natural inner-event completion leaves `m_timedEvent` holding an intrusive pointer, but `event_cancel()` explicitly handles `q_el == nullptr` and safely clears it; no UAF promoted.
- `Quiz(level)` uses `if (level > m_vec_quiz.size())` instead of `>=`; level==size can index out of range. Current deployed caller always requests level 1 and the deployed quiz table creates vector index 1, so this remains API-only/unpromoted.
- O/X answer rectangles leave a divider gap; no invariant currently proves it is unintended.
- map 113 is single-core in the tracked deployment.

## Open work
1. audit logout/relog and eliminated-player lifecycle beyond the cap-accounting path;
2. audit `Show()` based audience movement and map/state validation;
3. audit reward delivery/offline-winner behavior;
4. close automatic-start dependency on persisted level/max flags;
5. decide STATIC COMPLETE readiness.

## Runtime
No OX runtime test may be executed while the global execution lock is active. First future live gate remains `DUNGEON-T09`.
