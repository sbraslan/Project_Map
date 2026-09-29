# OX Event — Bug Registry

**Status:** STATIC MAPPING IN PROGRESS / 5 VERIFIED BUGS  
**Execution:** LOCKED / NOT RUN

## BUG-OX-001 — renewal quiz timer collides with the 35-second outer scheduler

**Class:** deterministic event scheduling / state-machine corruption  
**Reachability:** VERIFIED through active `ENABLE_EVENT_MANAGER + ENABLE_OX_RENEWAL`.

### Proof
1. Automatic OX starts questions from `ox_event_process`.
2. After `Quiz(1,30)`, the outer event returns `PASSES_PER_SEC(35)`.
3. Renewal inner timer timeline is:
   - +15s stage 0;
   - +20s stage 1 / `CheckAnswer`;
   - +35s stage 2 / `WarpToAudience` / status CLOSE / reset static flag.
4. The outer +35 event was queued before the inner stage-2 +35 requeue.
5. The queue uses `stable_priority_queue`; equal-key earlier entries are processed first.
6. Outer +35 therefore runs first and invokes another `Quiz`.
7. `Quiz` cancels the old `m_timedEvent` before its stage 2 executes.
8. `oxevent_timer` stores its stage in function-local `static uint8_t flag`, so cancellation does not reset it.
9. The next question's first inner callback sees stale flag=2, skips `CheckAnswer`, executes cleanup and resets to 0.

### Consequence
The automatic renewal OX cadence is structurally corrupted: every collision can cancel the previous cleanup and make the following question skip answer evaluation, while prior losers remain active until the next timer callback.

### Deferred validation
`OX-T01`.

## BUG-OX-002 — OX player-cap accounting is non-authoritative and can be corrupted

**Class:** admission race / distributed event flag  
**Reachability:** VERIFIED in normal multiplayer entry.

### Proof
1. Entry quest blocks only when `ox_map_login_counter == ox_map_player_max`.
2. The check occurs before the cross-core warp.
3. The counter is incremented only after map-113 login/enter.
4. `game.set_event_flag` only sends `HEADER_GD_SET_EVENT_FLAG`; it does not mutate the local flag immediately.
5. DB later broadcasts `HEADER_DG_SET_EVENT_FLAG` and only then do game cores update their local flag copies.
6. `COXEventManager::EnterAttender` has no authoritative max-count guard.
7. Two entrants can therefore pass while one slot remains.
8. Their target-side counter writes can either collapse to the same value or produce max+1.
9. If the counter becomes max+1, the quest's equality-only guard returns “allowed” again because it does not test `>=`.

10. A participant relogging at exact attendee spawn triggers the deployed map login handler again, increments the global counter again, but `m_map_attender.insert(pid,pid)` does not add another unique attendee.
11. The logout cooldown is checked only before the original NPC warp, not on a map relog.

### Consequence
Actual OX attendees can exceed the configured maximum; the counter can diverge from unique attendee count, and any state above max reopens entry because the quest checks equality rather than `>=`. A single participant can also consume/corrupt cap accounting by repeated spawn relogs.

### Deferred validation
`OX-T02`.

## BUG-OX-003 — players already warping during OPEN can become attendees after registration closes

**Class:** TOCTOU / registration state validation  
**Reachability:** VERIFIED at the normal registration boundary.

### Proof
1. NPC quest checks `oxevent_status == OPEN` before calling `pc.warp(896500,24600)`.
2. Map 113 is hosted on ch99/core99, so entry can require a cross-core transition.
3. Event Manager can change state to CLOSE while that transition is in flight.
4. Target login calls `COXEventManager::Enter(ch)`.
5. `Enter` rejects only `OXEVENT_FINISH`.
6. CLOSE and QUIZ therefore continue to the coordinate classifier.
7. Exact participant coordinates call `EnterAttender`, which unconditionally inserts the PID into attendee state.

### Consequence
A player admitted just before the source-side registration cutoff can arrive after the cutoff and still become an active quiz participant, including after quiz state has begun.

### Deferred validation
`OX-T03`.

## BUG-OX-004 — automatic round restart does not reset the deployed admission counter

**Class:** multi-round state reset / quest-manager integration  
**Reachability:** VERIFIED in the intended automatic three-round OX path.

### Proof
1. `OX_ROUND_COUNT = 3`.
2. Outer FINISH state with rounds remaining calls `COXEventManager::CloseEvent()`.
3. CloseEvent/Initialize clears local participant state.
4. Outer state is changed back to OPEN and registration countdown is reset.
5. No server OX path resets `ox_map_login_counter`.
6. The deployed quest resets that counter only in its manual `cleanup_event()` helper / force-management path.
7. New entrants still use `check_limit()` against the stale cumulative counter.

### Consequence
Later automatic rounds inherit earlier-round admission usage. A full first round can make round 2 registration reject everyone; partial rounds expose only the remaining cumulative slots.

### Deferred validation
`OX-T04`.

## BUG-OX-005 — GM force-end does not cancel the automatic Event Manager process

**Class:** scheduler ownership / stop semantics  
**Reachability:** VERIFIED when an Event Manager OX is administratively force-ended through the deployed GM quest.

### Proof
1. Event Manager owns outer `m_pOXEvent`.
2. Deployed GM force-end calls only `oxevent.end_event_force()`.
3. Lua binding executes `COXEventManager::CloseEvent()` and sets FINISH.
4. It never calls `CEventManager::SetOXEvent(false)` or cancels `m_pOXEvent`.
5. The outer process remains queued and retains its own state/round counters.
6. Under `ENABLE_EVENT_MANAGER`, COX Initialize does not clear the quiz vector.
7. A later outer callback can continue and set status/launch quiz logic again.

### Consequence
An OX event reported/forced as ended can partially resurrect from its still-live automatic scheduler.

### Deferred validation
`OX-T05`.

## Open candidates
- Automatic Event Manager OX does not initialize the deployed quest's `ox_map_level_min/max`, `ox_map_player_max`, or login counter; current DB event-table/flag values are required before declaring this deployed breakage.
- Automatic multi-round restart clears COX local state but does not explicitly reset `ox_map_login_counter`; interaction with current persisted flag values remains under audit.
- `Quiz(level == m_vec_quiz.size())` can index out of bounds, but deployed quiz caller/table do not reach it.
