# Monarch — Bug Registry

**Status:** STATIC MAPPING IN PROGRESS / 4 VERIFIED BUGS  
**Execution:** LOCKED / NOT RUN

## BUG-MON-001 — election finalizer does not produce a monarch

**Class:** election state machine / uninitialized counter / wrong identity  
**Reachability:** VERIFIED through registered `elect` GM command.

### Proof
1. `elect` sends `HEADER_GD_COME_TO_VOTE`.
2. DB directly calls `CMonarch::ElectMonarch()`.
3. It allocates `int* s = new int[size]` without value initialization.
4. Context7/cppreference confirms fundamental elements are indeterminate in this form.
5. Election records contain voter `pid` and `selectedpid`.
6. Counting calls `GetCandidacyIndex(it->second->pid)`, using voter PID instead of selected candidate PID.
7. After increments, the array is immediately deleted.
8. There is no winner comparison, `SetMonarch`, SQL winner mutation or monarch-info broadcast.

### Consequence
The registered election-finalization command cannot elect or publish a winner; even its internal count is invalid.

### Deferred validation
`MON-T01`.

## BUG-MON-002 — setmonarch does not refresh live monarch state

**Class:** DB/game replication / wrong packet header  
**Reachability:** VERIFIED through registered `setmonarch` GM command.

### Proof
1. DB `SetMonarch` mutates its in-memory state and issues a persistent SQL replace.
2. On success `CClientManager::SetMonarch` broadcasts `HEADER_DG_RMCANDIDACY`.
3. It does not broadcast `HEADER_DG_UPDATE_MONARCH_INFO`.
4. Game `CInputDB::Analyze` has no SETMONARCH/RMCANDIDACY handling.
5. The known-good `ChangeMonarchLord` path explicitly reloads and broadcasts `HEADER_DG_UPDATE_MONARCH_INFO`.

### Consequence
A GM can change monarch persistence while active game cores continue using the old monarch PID/name/money state until a later proper refresh/restart.

### Deferred validation
`MON-T02`.

## BUG-MON-003 — rmmonarch double-deletes and loses successful removal fanout

**Class:** administrative mutation / state replication  
**Reachability:** VERIFIED through registered `rmmonarch` GM command.

### Proof
`CClientManager::RMMonarch` calls `DelMonarch(szName)` once and discards the result, then immediately calls it again to calculate `iRet`.

If the first call succeeds, the matching in-memory monarch identity has already been cleared, so the second name lookup cannot succeed. The success fanout branch is therefore not reached for that successful first deletion.

No authoritative `HEADER_DG_UPDATE_MONARCH_INFO` is emitted.

### Consequence
Persistent/in-DB monarch removal can occur without live game cores receiving the corresponding state removal.

### Deferred validation
`MON-T03`.

## BUG-MON-004 — invalid monarch tax is applied after being rejected

**Class:** validation/control-flow  
**Reachability:** VERIFIED through registered player-level `mtax` command gated by `IsMonarch()`.

### Proof
1. Command usage and check define valid range 1..50.
2. For values outside range, code sends an error message only.
3. There is no return.
4. It proceeds to `SetEventFlag("trade_tax", tax)`.
5. It broadcasts notices with the invalid percentage and sets the monarch cooldown.

### Consequence
A monarch can write out-of-range tax state despite being told the value is invalid.

### Deferred validation
`MON-T04`.

## Open candidates
- treasury multi-core race;
- process-local power/defense buffs;
- unguarded `takemonarchmoney` Lua API while security block is compiled under `__UNIMPLEMENTED__`;
- warp/transfer charge-on-failure ordering;
- SetMonarch SQL/schema mismatch;
- DelMonarch DELETE-result interpretation.
