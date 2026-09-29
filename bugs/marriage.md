# Marriage / Wedding — Bug Registry

**Status:** STATIC COMPLETE / 8 VERIFIED BUGS  
**Execution:** LOCKED / NOT RUN

## BUG-MARR-001 — engagement resource transaction commits before authoritative marriage creation

**Class:** transaction ordering / state consistency  
**Reachability:** VERIFIED through deployed `marriage_manage.quest`.

### Proof
1. The deployed engagement quest validates both characters and waits for `confirm(u_vid,...)`.
2. On `CONFIRM_OK` it immediately deducts 1,000,000 Yang from the initiator.
3. It removes engagement ring 70301 from both characters and gives marriage ring 70302 to both.
4. It then executes dialogue followed by `wait()`, suspending the quest.
5. Only after that suspension does it call `marriage.engage_to(u_vid)`.
6. The registered Lua binding resolves the target using `CHARACTER_MANAGER::Find(vid)`.
7. If the target cannot be resolved, the binding returns without `CManager::RequestAdd`.
8. No rollback restores the money/items already committed.

### Consequence
The engagement transaction can leave resources/rings changed while no engagement/marriage row is created.

### Deferred validation
`MARR-T01`.

## BUG-MARR-002 — one wedding request can create private wedding maps on two deployed cores

**Class:** deployment topology / duplicate producer / lifecycle consistency  
**Reachability:** VERIFIED against current tracked CONFIG files.

### Proof
1. `MAP_WEDDING_01 = 81`.
2. Current deployment enables map 81 on both:
   - `chan/ch2/core4/CONFIG`;
   - `chan/ch99/core99/CONFIG`.
3. Game marriage creation sends `HEADER_GD_WEDDING_REQUEST`.
4. DB `CClientManager::WeddingRequest` uses `ForwardPacket(HEADER_DG_WEDDING_REQUEST,...)` with no channel restriction.
5. `ForwardPacket` iterates every connected game peer with a non-zero channel.
6. Every recipient calls `WeddingManager::Request`; every core where `map_allow_find(81)` is true can create a private wedding map.
7. Each producer sends `HEADER_GD_WEDDING_READY`.
8. DB `CManager::ReadyWedding` queues every READY with no pair-level deduplication.
9. Game `CManager::WeddingReady` stores one `pWeddingInfo->dwMapIndex`; a later READY overwrites the earlier index.
10. `TPacketWeddingStart` contains only PID1/PID2 and cannot identify which READY/map instance was selected.

### Consequence
A single wedding can produce duplicate map instances and duplicate scheduling while game-side relation state retains only one last-delivered map index, leaving ambiguous/orphaned wedding lifecycle state.

### Deferred validation
`MARR-T02`.

## BUG-MARR-003 — mutual divorce rejects an exactly sufficient 500,000 Yang balance

**Class:** boundary condition / gameplay availability  
**Reachability:** VERIFIED through deployed `marriage_manage.quest`.

### Proof
- Mutual divorce defines `MONEY_NEED_FOR_ONE = 500000`.
- Initial checks use `pc.gold > MONEY_NEED_FOR_ONE` and partner `pc.get_gold() > MONEY_NEED_FOR_ONE`.
- Post-confirm checks repeat the same strict-greater comparison.
- Successful processing deducts exactly `MONEY_NEED_FOR_ONE`.

### Consequence
A player with exactly 500,000 Yang has the exact required fee but is rejected as insufficient, blocking mutual divorce until the balance exceeds the documented/debited amount.

### Deferred validation
`MARR-T03`.

## BUG-MARR-004 — stale old-core P2P logout can force a false lover-offline state after a fresh login

**Class:** cross-core ordering / session state desynchronization  
**Reachability:** VERIFIED from the existing name-keyed CCI handoff race plus Marriage logout integration.

### Proof
1. `P2P_MANAGER::Login` reuses an existing CCI by name and updates its `pkDesc`/channel/map to the newest login.
2. `CInputP2P::Logout` discards its source descriptor and calls `P2P_MANAGER::Logout(p->szName)`.
3. Name-only logout removes whichever CCI is current for that name, even if the packet came from the previous core.
4. `P2P_MANAGER::Logout(CCI*)` always calls `marriage::CManager::Logout(pid)`.
5. Married `TMarriage::Logout` emits `lover_logout` to available local/relayed spouse descriptors.
6. Client `game.py::__LogoutLover` calls messenger `OnLogoutLover()` and hides lover state.

### Consequence
After LOGIN(new) has already established the fresh state, delayed LOGOUT(old) can mark the spouse offline again. Because the fresh login occurred first, no later corrective lover-login event is guaranteed.

### Deferred validation
`MARR-T04`.

## BUG-MARR-005 — ExitToSavedLocation ignores the actual saved exit position

**Class:** warp/lifecycle / wrong state field  
**Reachability:** VERIFIED through normal wedding entry and end.

### Proof
1. Wedding warp calls `SaveExitLocation()`.
2. `SaveExitLocation()` stores current coordinates/map in `m_posExit` and `m_lExitMapIndex`.
3. After normal warp completion, `WarpEnd()` clears `m_posWarp` and `m_lWarpMapIndex`.
4. Wedding end calls `WeddingMap::WarpAll()`.
5. `WarpAll()` calls `CHARACTER::ExitToSavedLocation()`.
6. `ExitToSavedLocation()` incorrectly calls `WarpSet(m_posWarp.x, m_posWarp.y, m_lWarpMapIndex)`, not the saved exit fields.
7. It then clears `m_posExit/m_lExitMapIndex`.

### Consequence
The normal wedding shutdown path does not warp players back to the pre-wedding location saved for that purpose and destroys the only stored exit location afterward.

### Deferred validation
`MARR-T05`.

## BUG-MARR-006 — same-core wedding exit leaves stale membership and later destroys the already-exited player

**Class:** membership lifecycle / delayed teardown  
**Reachability:** VERIFIED for same-process exit warps.

### Proof
1. Wedding membership is removed only through `SetWeddingMap(nullptr) -> WeddingMap::DecMember`.
2. `WarpSet()` does not clear `m_pWeddingMap` or call `DecMember`.
3. End step 0 executes `WarpAll() -> ExitToSavedLocation()/WarpSet()`.
4. End step 1 runs 15 seconds later and calls `DestroyWeddingMap() -> DestroyAll()`.
5. `DestroyAll()` destroys every character still present in `m_set_pkChr`.
6. A same-process warp keeps the CHARACTER alive, so without explicit detachment it remains in that set.
7. Deployed map ownership allows same-core source destinations on map-81 hosting cores.

### Consequence
A player who successfully leaves the wedding map via a same-core warp can still be destroyed/disconnected by the wedding teardown 15 seconds later.

### Deferred validation
`MARR-T06`.

## BUG-MARR-007 — level-26+ EXP love-point progression is truncated to zero

**Class:** arithmetic / progression  
**Reachability:** VERIFIED through normal EXP distribution for married players.

### Proof
1. The deployed marriage quest requires level 25 or higher.
2. EXP distribution computes:
   `static_cast<uint32_t>(2000.0L / level / level / 3) * static_cast<uint32_t>(iFinalExp)`.
3. The fractional coefficient is converted to integer before multiplying by EXP.
4. At level 25 the coefficient is >1 and truncates to 1.
5. At level 26 and every higher level the coefficient is <1 and truncates to 0.
6. `TMarriage::Update` ignores zero, so no love-point progress/save flag is produced from that EXP.

### Consequence
Almost every normally progressing married character above level 25 loses the EXP-based love-point progression path completely; only time-based contribution remains.

### Deferred validation
`MARR-T07`.

## BUG-MARR-008 — marriage item effects are not shared across game cores

**Class:** distributed state / gameplay effect parity  
**Reachability:** VERIFIED for spouses online on different game cores.

### Proof
1. Deployed locale descriptions for 71069..71074 say that if one spouse equips the item, its effect applies to both spouses.
2. `CHARACTER::GetMarriageBonus` calls `TMarriage::GetBonus`.
3. Shared mode checks `ch1/ch2->IsEquipUniqueItem(vnum)`.
4. `ch1/ch2` are local CHARACTER pointers populated by Marriage `Login(ch)`.
5. P2P login does not populate a remote spouse CHARACTER pointer in Marriage state.
6. On separate cores, each process can inspect only its local spouse's equipment.

### Consequence
If only one spouse wears a marriage bonus item, the remote spouse on another core does not receive the advertised shared effect. Moving both spouses onto the same core can change the effect without any marriage/item change.

### Deferred validation
`MARR-T08`.

## Open candidates
- `WeddingManager::__CreateWeddingMap` inserts a WeddingMap/private map before checking whether `GetMap(dwMapIndex)` succeeds; the failure return currently has no cleanup. Keep unpromoted until realistic failure reachability is established.
- Several marriage Lua helpers trust quest-side state and dereference relation/wedding state with limited local validation. Audit all deployed callers before promotion.
- Mutual divorce confirm/reselection ordering still requires event-resume semantics closure before any stale-VID conclusion.
