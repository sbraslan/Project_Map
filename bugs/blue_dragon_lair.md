# Blue Dragon / Beran Setaou — Bug Registry

**Status:** STATIC MAPPING OPEN / 3 VERIFIED BUGS
**Execution:** LOCKED / NOT RUN

## BUG-BDL-001 — access items are consumed before a disconnect-cancellable personal entry timer

**Class:** item transaction / disconnect rollback

### Proof
- Entry first calls `pc.remove_item(access_item, access_item_amount)`.
- The first entrant then schedules the personal timer `dragon_lair_warptimer` with a delay of `pc.get_channel_id() * 2`.
- Successful start/warp and the competing-start refund are both implemented only inside that timer callback.
- Character disconnect eventually calls `CQuestManager::DisconnectPC`, which erases the player's quest `PC` object.
- `PC::~PC -> Destroy -> ClearTimer` cancels all personal quest timers.
- The Blue Dragon quest has no logout/disconnect compensation for the already removed entry items.

### Consequence
Disconnecting or losing the connection during the delayed entry window can permanently consume the Blue Dragon access items without starting/joining the run and without refund.

### Deferred validation
`BDL-T01`.



## BUG-BDL-002 — disconnect during locked entry dialogue leaves the Blue Dragon NPC locked to a dead PID

**Class:** quest lifecycle / availability denial

### Proof
- The first-entry flow acquires the NPC with `npc.lock()`, which stores the player's PID in the NPC's quest-lock field.
- `npc.unlock()` is called on ordinary abort/success branches.
- Normal quest completion also has a safety unlock in `PC::EndRunning()`.
- Character logout while a quest is suspended instead executes `CQuestManager::LogoutPC`: `CloseState` followed by `CancelRunning`.
- `CloseState` only unreferences the Lua coroutine, and `CancelRunning` only nulls running-state metadata.
- `PC::EndRunning()` is not called on that disconnect path.
- `npc.lock()` grants a lock only when `GetQuestNPCID()==0` or already equals the requesting PID; it does not validate whether the stored PID is still connected.

### Consequence
Disconnecting while the Blue Dragon entry NPC is locked can strand the NPC with the departed player's PID, preventing other players from starting the lair until that NPC is recreated/reset or the lock is manually cleared.

### Deferred validation
`BDL-T02`.


## BUG-BDL-003 — existing-run join ignores that Beran has already died

**Class:** run-state validation / paid entry after completion

### Proof
- The existing-run entry condition is `starttime + group_time >= current_time`.
- That branch validates the shared entry code and access items, consumes the items, stores the run start time and warps the player into map 208.
- It never checks `dragon_lair_alive`.
- The 2493 kill handler sets `dragon_lair_alive = 0` and purges the room but does not clear `dragon_lair_time`.
- `group_time` is half the 20-minute live cooldown, so a kill inside the first 10 minutes leaves a positive paid-join window for an already completed run.

### Consequence
Players can spend Blue Dragon access items to enter an empty/completed lair after Beran has already been killed.

### Deferred validation
`BDL-T03`.
