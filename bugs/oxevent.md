# OX Event — Bug Registry

**Status:** STATIC MAPPING IN PROGRESS / 3 VERIFIED BUGS  
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

## BUG-OX-002 — asynchronous login counter cannot enforce the configured player cap atomically

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

### Consequence
Actual OX attendees can exceed the configured maximum; under max+1 state the quest can also reopen further admissions.

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

## Open candidates
- Automatic Event Manager OX does not initialize the deployed quest's `ox_map_level_min/max`, `ox_map_player_max`, or login counter; current DB event-table/flag values are required before declaring this deployed breakage.
- Automatic multi-round restart clears COX local state but does not explicitly reset `ox_map_login_counter`; interaction with current persisted flag values remains under audit.
- `Quiz(level == m_vec_quiz.size())` can index out of bounds, but deployed quiz caller/table do not reach it.
