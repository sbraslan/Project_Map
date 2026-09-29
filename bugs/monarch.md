# Monarch — Bug Registry

**Status:** STATIC MAPPING IN PROGRESS / 7 VERIFIED BUGS  
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

## BUG-MON-003 — rmmonarch DELETE succeeds but result handling forces failure/stale state

**Class:** SQL result handling / administrative state replication  
**Reachability:** VERIFIED through registered `rmmonarch` GM command.

### Proof
1. `DelMonarch` issues `DELETE FROM monarch WHERE empire=...`.
2. Project SQL wrapper stores affected-row count in `uiAffectedRows`.
3. `uiNumRows` is populated only from a result set returned by `mysql_store_result`; for DELETE it remains 0.
4. `DelMonarch` tests `uiNumRows == 0` and returns false.
5. It therefore does not clear `m_MonarchInfo` even if the row was actually deleted.
6. `RMMonarch` calls `DelMonarch` twice and then follows its failure branch.
7. No `HEADER_DG_UPDATE_MONARCH_INFO` is broadcast.

### Consequence
The persistent monarch row can be deleted while DB/game runtime still believes the old monarch exists; the admin command reports/fanouts failure rather than reconciling state.

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


## BUG-MON-005 — stale local treasury prechecks can grant an unpaid monarch effect

**Class:** async transaction ordering / treasury consistency  
**Reachability:** VERIFIED through registered `mtr` and `mmob` commands.

### Proof
1. Treasury prechecks use each game core's cached `TMonarchInfo.money`.
2. `SendtoDBDecMoney` sends an async GD request and does not reserve/decrement local balance.
3. `mtr` and `mmob` use distinct cooldown slots, MI_TRANSFER and MI_SUMMON.
4. Their gameplay effects are applied/sent before authoritative DB deduction is confirmed.
5. A second command can therefore pass against the same old balance before the first DG delta returns.
6. DB `DecMonarchMoney` ignores the return value of authoritative `CMonarch::DecMoney`.
7. It broadcasts a DEC packet even if the DB deduction was rejected for insufficient funds.
8. There is no rollback/failure ACK for the gameplay effect.

### Consequence
At a treasury boundary, one of two rapid different monarch actions can complete without an authoritative treasury charge.

### Deferred validation
`MON-T05`.

## BUG-MON-006 — MI_TAX cooldown is never checked by mtax

**Class:** cooldown enforcement  
**Reachability:** VERIFIED through registered `mtax` command.

### Proof
- `InitMC` defines a cooldown limit for MI_TAX.
- `do_monarch_tax` calls `SetMC(MI_TAX)`.
- The command contains no `IsMCOK(MI_TAX)` precheck.
- Other monarch commands explicitly check their corresponding cooldown before acting.

### Consequence
A monarch can repeatedly change tax without respecting the configured tax cooldown.

### Deferred validation
`MON-T06`.

## BUG-MON-007 — remote mtr charges before knowing whether target still exists

**Class:** P2P race / charge-on-failure  
**Reachability:** VERIFIED through registered `mtr` remote-target path.

### Proof
1. Source core finds a remote target CCI and sends `HEADER_GG_TRANSFER`.
2. Source immediately sends treasury DEC request and sets MI_TRANSFER cooldown.
3. Receiving core resolves target by name at packet handling time.
4. If target is absent, receiver performs no warp and sends no failure response.
5. No transfer success acknowledgement exists.

### Consequence
A disconnect/core-handoff race can consume treasury and cooldown while the requested transfer never occurs.

### Deferred validation
`MON-T07`.
