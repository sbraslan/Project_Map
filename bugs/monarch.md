# Monarch — Bug Registry

**Status:** STATIC MAPPING IN PROGRESS / 9 VERIFIED BUGS  
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

## BUG-MON-005 — asynchronous treasury deduction permits effect-before-payment race

**Class:** treasury concurrency / stale cached balance  
**Reachability:** VERIFIED through registered monarch actions.

### Proof
1. Each game core caches `TMonarchInfo.money`.
2. `SendtoDBDecMoney` checks the cached balance but does not reserve/decrement it.
3. Actions such as transfer/warp/summon apply or dispatch their gameplay effect before the DB response arrives.
4. Distinct monarch actions use distinct cooldown categories, so one action does not serialize all treasury spending.
5. Two requests can therefore pass the same cached balance before either DG decrement updates the core.
6. DB `CMonarch::DecMoney` can reject the later request if authoritative money is insufficient.
7. `CClientManager::DecMonarchMoney` ignores that boolean result and broadcasts the decrement packet anyway.
8. The already-triggered gameplay effect has no rollback.

### Consequence
A monarch can receive multiple treasury-backed effects while authoritative persistence pays for fewer of them under a tight asynchronous request window.

### Deferred validation
`MON-T05`.

## BUG-MON-006 — setmonarch persists PID to `name` instead of `pid`

**Class:** persistence/schema contract  
**Reachability:** VERIFIED through registered `setmonarch` GM command.

### Proof
1. DB resolves the selected player and stores the correct ID in `m_MonarchInfo.pid[Empire]`.
2. It persists:
   `REPLACE INTO monarch (empire, name, windate, money) VALUES(..., p->pid[Empire], ...)`.
3. It does not write column `pid`.
4. `LoadMonarch` reconstructs identity from `a.pid` and joins `a.pid=b.id`.
5. The newer `ChangeMonarchLord` path correctly executes `UPDATE monarch SET pid=...`.
6. The REPLACE is asynchronous and its failure/result is not checked.

### Consequence
The legacy GM set path can establish an in-memory monarch while failing to persist the selected identity in the column used at the next authoritative reload.

### Deferred validation
`MON-T06`.

## BUG-MON-007 — cross-core monarch transfer is charged without delivery acknowledgement

**Class:** P2P lifecycle / transaction ordering  
**Reachability:** VERIFIED through registered `mtr` player command for a valid monarch.

### Proof
1. Source core resolves a remote target through P2P CCI and validates cached empire/channel/map state.
2. It broadcasts `HEADER_GG_TRANSFER`.
3. It immediately sends the treasury deduction request and sets `MI_TRANSFER` cooldown.
4. Remote `CInputP2P::Transfer` looks up the target by name.
5. If the target is absent it silently does nothing.
6. If present, it calls `WarpSet` without returning success to the source.
7. There is no P2P transfer ACK and no charge/cooldown rollback.

### Consequence
A target logout/core handoff or warp failure after the source-side CCI check can consume treasury funds and cooldown while no transfer occurs.

### Deferred validation
`MON-T07`.

## BUG-MON-008 — election SQL survives DB restart but election runtime state does not

**Class:** persistence reconstruction  
**Reachability:** VERIFIED.

### Proof
1. `VoteMonarch` inserts votes into `monarch_election`.
2. `AddCandidacy` inserts candidates into `monarch_candidacy`.
3. Active election logic reads only `m_map_MonarchElection` and `m_vec_MonarchCandidacy`.
4. `CMonarch` construction starts those containers empty.
5. DB startup `InitializeMonarch()` calls only `LoadMonarch()`.
6. `LoadMonarch()` selects only from the `monarch` table.
7. No candidacy/election reload method exists in the class.
8. Game boot receives whatever is currently in the empty candidate vector.

### Consequence
A DB restart during an election disconnects persistent vote/candidate rows from the runtime election state; finalization and candidate administration operate on an empty/rebuilt-incompletely state.

### Deferred validation
`MON-T08`.

## BUG-MON-009 — monarch action cooldowns reset on character recreation

**Class:** session lifecycle / cooldown persistence  
**Reachability:** VERIFIED for commands/functions that check `IsMCOK`.

### Proof
1. `CHARACTER::Initialize()` calls `InitMC()`.
2. `InitMC()` sets each Monarch cooldown timestamp to current pulse.
3. It immediately subtracts that category's full limit, making `IsMCOK` true.
4. Cooldown fields are members of the transient `CHARACTER` object.
5. The mapped player persistence structure contains no Monarch cooldown fields.

### Consequence
Logging out and creating a fresh character session clears active cooldowns for heal, warp, transfer and summon. Tax is already separately affected by `BUG-MON-006` because its command never checks `MI_TAX`.

### Deferred validation
`MON-T09`.

## Open candidates
- process-local PowerUp/DefenseUp buffs; deployed caller closure pending;
- unguarded `takemonarchmoney` Lua API while validation is compiled under `__UNIMPLEMENTED__`; deployed caller not yet proven;
- add-money overflow/failure reporting symmetry;
- legacy `SetMonarch` SQL column mismatch remains candidate until authoritative table schema is available.
