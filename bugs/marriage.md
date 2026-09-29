# Marriage / Wedding — Bug Registry

**Status:** STATIC MAPPING IN PROGRESS / 2 VERIFIED BUGS  
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

## Open candidates
- `WeddingManager::__CreateWeddingMap` inserts a WeddingMap/private map before checking whether `GetMap(dwMapIndex)` succeeds; the failure return currently has no cleanup. Keep unpromoted until realistic failure reachability is established.
- Several marriage Lua helpers trust quest-side state and dereference relation/wedding state with limited local validation. Audit all deployed callers before promotion.
- Mutual divorce confirm/reselection ordering still requires event-resume semantics closure before any stale-VID conclusion.
