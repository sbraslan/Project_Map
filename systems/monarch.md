# Monarch — Static Map

**Status:** STATIC MAPPING IN PROGRESS / 12 VERIFIED BUGS  
**Phase:** Detection / Mapping Only  
**Opened:** 2026-09-29  
**Source/Game repositories:** READ-ONLY  
**Runtime:** LOCKED / NOT RUN

## Scope
- DB monarch persistence and game-core state replication;
- GM set/remove/election administration;
- treasury add/decrease/take paths;
- monarch player commands (warp, transfer, tax, summon, notice);
- Lua monarch powers and cooldowns;
- multi-core state consistency.

## Main roots
- `Project_ServerSRC/db/src/Monarch.cpp`
- `Project_ServerSRC/db/src/Monarch.h`
- `Project_ServerSRC/db/src/ClientManager.cpp`
- `Project_ServerSRC/db/src/ClientManagerBoot.cpp`
- `Project_ServerSRC/game/src/monarch.cpp`
- `Project_ServerSRC/game/src/monarch.h`
- `Project_ServerSRC/game/src/questlua_monarch.cpp`
- `Project_ServerSRC/game/src/cmd.cpp`
- `Project_ServerSRC/game/src/cmd_general.cpp`
- `Project_ServerSRC/game/src/cmd_gm.cpp`
- `Project_ServerSRC/game/src/input_db.cpp`
- `Project_ServerSRC/game/src/char_battle.cpp`
- `Project_ServerSRC/common/tables.h`

## Boot/state model
DB boot calls `CMonarch::LoadMonarch()`.

Each game core receives `TMonarchInfo` in the boot packet and stores it through:
`CInputDB::Boot -> CMonarch::SetMonarchInfo`.

The normal live state-refresh path is:
DB mutation
→ reload DB singleton if needed
→ `HEADER_DG_UPDATE_MONARCH_INFO`
→ game `CInputDB::UpdateMonarchInfo`
→ `CMonarch::SetMonarchInfo`.

`ChangeMonarchLord` uses this correct pattern.

## BUG-MON-001 — election finalizer cannot elect a winner
Registered GM command:
`elect` (GM_HIGH_WIZARD)
→ `HEADER_GD_COME_TO_VOTE`
→ DB `CClientManager::ComeToVote`
→ `CMonarch::ElectMonarch()`.

`ElectMonarch()`:
- allocates `new int[size]`;
- never zero-initializes the scalar array;
- iterates election records;
- looks up candidacy using `it->second->pid` (the voter PID), not `selectedpid`;
- increments that index if found;
- deletes the array;
- never selects a maximum, never calls `SetMonarch`, and never emits a winner/state update.

Context7/cppreference confirms that `new int[n]` without value initialization leaves fundamental elements indeterminate.

Promoted as `BUG-MON-001`.

## BUG-MON-002 — setmonarch persists DB state but does not synchronize live game cores
Registered GM command:
`setmonarch <name>` (GM_LOW_WIZARD)
→ `HEADER_GD_SETMONARCH`
→ DB `CClientManager::SetMonarch`
→ DB `CMonarch::SetMonarch`.

On success DB broadcasts `HEADER_DG_RMCANDIDACY`, not `HEADER_DG_SETMONARCH` or `HEADER_DG_UPDATE_MONARCH_INFO`.

Game `CInputDB::Analyze` has no handler for SETMONARCH/RMCANDIDACY responses. Runtime monarch state is therefore not updated. The correct live-update mechanism exists separately in `ChangeMonarchLord`.

Promoted as `BUG-MON-002`.

## BUG-MON-003 — rmmonarch deletes persistence but always reports failure and leaves runtime state stale
Registered GM command:
`rmmonarch <name>`
→ `HEADER_GD_RMMONARCH`
→ DB `CClientManager::RMMonarch`.

`CMonarch::DelMonarch` executes a SQL `DELETE`, then tests `pMsg->Get()->uiNumRows == 0`.

The project's SQL wrapper sets:
- `uiAffectedRows = mysql_affected_rows(...)`;
- `uiNumRows = mysql_num_rows(...)` only when `mysql_store_result` returns a result set;
- otherwise `uiNumRows = 0`.

A DELETE has no result-set rows, so a successful DELETE still leaves `uiNumRows == 0`. `DelMonarch` therefore returns false before clearing its in-memory monarch fields.

`CClientManager::RMMonarch` additionally calls `DelMonarch(szName)` twice. Neither path emits an authoritative `HEADER_DG_UPDATE_MONARCH_INFO`.

Promoted as `BUG-MON-003`.

## BUG-MON-004 — mtax validation reports invalid input but still applies it
Registered player command:
`mtax` (GM_PLAYER)
→ `do_monarch_tax`.

The command correctly requires `ch->IsMonarch()` and documents range 1..50.

For `tax < 1 || tax > 50` it only sends an error message, but does not return. It then executes:
`SetEventFlag("trade_tax", tax)`,
broadcasts notices using the invalid value, and sets the monarch tax cooldown.

Promoted as `BUG-MON-004`.

## Treasury flow
Game requests:
- `HEADER_GD_ADD_MONARCH_MONEY`
- `HEADER_GD_DEC_MONARCH_MONEY`
- `HEADER_GD_TAKE_MONARCH_MONEY`

DB owns persisted treasury value and broadcasts add/decrease deltas. Game cores maintain local copies in `TMonarchInfo`.

Open audit: concurrent multi-core operations can pass stale local `IsMoneyOk` checks before DB serializes deductions; effects are often applied before/without an explicit DB success acknowledgement.

## Power/Defense model
`monarchpowerup` / `monarchdefenseup` set process-local arrays:
- `m_PowerUp[4]`
- `m_DefenseUp[4]`

Combat reads these arrays directly in `char_battle.cpp` for ±10% damage.

No DB/P2P replication was found in the mapped path. Keep as a candidate until a currently deployed quest/caller for these Lua functions is proven.

## Early candidates (later closure below)
- PowerUp/DefenseUp and `takemonarchmoney` were audited for deployed callers; current tracked quest corpus contains no caller.
- Add-money overflow/failure symmetry remains in final boundary pass.
- Monarch cooldown reconstruction was promoted as `BUG-MON-009`.

## Earlier audit cursor
The items below were subsequently advanced in the later sections of this file; the canonical resume cursor is the final `Current audit cursor` at the end.

## Runtime
No Monarch runtime/fault-injection test may execute while the global execution lock is active. First future live gate remains `DUNGEON-T09`.


## BUG-MON-005 — asynchronous treasury deductions can grant a monarch effect that DB refuses to charge
Registered monarch commands use game-core-local cached treasury state for prechecks, but the cache is not reduced when a GD deduction request is sent.

Concrete reachable pair:
1. `mtr` checks local balance >= 10,000, performs/sends the transfer, sends `HEADER_GD_DEC_MONARCH_MONEY`, and sets MI_TRANSFER.
2. Before the DG deduction returns, `mmob` has an independent MI_SUMMON cooldown and can still see the same old local balance.
3. `mmob` can spawn the monster immediately, then send a 5,000,000 deduction.

With a treasury balance at the boundary, both effects can pass local checks while DB can fund only one deduction.

DB `CClientManager::DecMonarchMoney` ignores the bool returned by DB `CMonarch::DecMoney` and broadcasts the requested DG DEC delta even when the authoritative deduction failed.

Game-core `DecMoney` can refuse the underflow locally, but there is no failure acknowledgement capable of rolling back the already-applied transfer/spawn effect.

Promoted as `BUG-MON-005`.

## BUG-MON-006 — setmonarch persistence column differs from authoritative loader
Legacy `CMonarch::SetMonarch` updates the selected PID in memory, then persists:
`REPLACE INTO monarch (empire, name, windate, money) VALUES(..., p->pid[Empire], ...)`.

It omits `pid`, while `LoadMonarch` reconstructs identity from `a.pid` and joins `a.pid=b.id`. The newer `ChangeMonarchLord` path explicitly writes `pid`.

No tracked schema/trigger path repairs this source-level column contract mismatch, and the REPLACE result is asynchronous/unverified.

Promoted as `BUG-MON-006`.

## BUG-MON-007 — cross-core monarch transfer charges and consumes cooldown without delivery acknowledgement
Registered `mtr` can target a remote player tracked by P2P CCI.

The source core:
- validates the CCI;
- sends `TPacketGGTransfer`;
- immediately requests 10,000 treasury deduction;
- immediately sets MI_TRANSFER cooldown.

Receiver `CInputP2P::Transfer` simply performs:
`FindPC(name)`, and only if the target still exists does it call `WarpSet`.

There is no ACK/NACK to the sender. If the target disconnects or changes ownership between CCI lookup and P2P handling, the receiver silently does nothing while the source has already charged treasury and consumed cooldown.

Promoted as `BUG-MON-007`.

## Caller/deployment closure
A recursive tracked quest inventory plus repository search found no current `Project_Game` caller for:
- `oh.takemonarchmoney`
- `oh.monarchpowerup`
- `oh.monarchdefenseup`
- `oh.monarchbless`.

These Lua APIs remain mapped but are not promoted as deployed gameplay bugs without a tracked caller.


## BUG-MON-008 — DB restart discards persisted candidacy and vote state from runtime

**Class:** restart reconstruction / election lifecycle  
**Reachability:** VERIFIED.

Candidacy and voting are persisted to:
- `monarch_candidacy`
- `monarch_election`

But DB startup `CClientManager::InitializeMonarch()` calls only:
`CMonarch::LoadMonarch()`.

`LoadMonarch()` reads only the `monarch` table. The class has no loader for candidacy or election tables, and its constructor begins with empty `m_vec_MonarchCandidacy` and `m_map_MonarchElection`.

After DB restart, persisted candidate/vote rows therefore remain in SQL but are absent from the in-memory election state used by `ElectMonarch`, `IsCandidacy` and boot candidacy fanout.

Promoted as `BUG-MON-008`.

## BUG-MON-009 — relog recreates all enforced monarch cooldowns as immediately ready

**Class:** session lifecycle / cooldown persistence  
**Reachability:** VERIFIED for monarch actions that use `IsMCOK`.

Every new `CHARACTER` object calls `Initialize() -> InitMC()`.

`InitMC()` initializes each cooldown timestamp to the current pulse and then subtracts the complete cooldown limit, intentionally making every category immediately usable:
- heal;
- warp;
- transfer;
- tax slot;
- summon.

These timestamps are not part of `TPlayerTable` or another mapped persistence path.

A logout/login or character-object recreation therefore loses active cooldown history. This is independent of `BUG-MON-006`, where the tax command does not check its cooldown at all.

Promoted as `BUG-MON-009`.

## Remaining closure work
- close PowerUp/DefenseUp and `takemonarchmoney` as deployed versus dormant;
- inspect add-money edge handling;
- inspect monarch notice and remaining warp edges;
- decide STATIC COMPLETE.


## BUG-MON-010 — MI_TAX cooldown is not enforced
`do_monarch_tax` sets `MI_TAX` after changing tax state but does not call `IsMCOK(MI_TAX)` before accepting the next change. Other monarch actions enforce their cooldown categories.

Promoted as `BUG-MON-010`.

## Dormant candidate closure
- `oh.monarchpowerup`, `oh.monarchdefenseup`, `oh.monarchbless` and `oh.takemonarchmoney` have no current tracked Project_Game caller.
- `monarch_defenseup_event` has a wrong dynamic_cast target type and would risk a stuck defense effect if that Lua API becomes deployed; kept dormant/unpromoted under current corpus.
- `takemonarchmoney` security checks remain under `__UNIMPLEMENTED__`, but no deployed caller was found.

## Current audit cursor
1. audit AddMoney overflow/failure fanout symmetry;
2. audit remaining mto/mtr private-map and WarpSet failure boundaries;
3. inspect notice path and treasury producers for independent defects;
4. decide Monarch STATIC COMPLETE.


## BUG-MON-011 — mto loses private-instance identity
`IsMonarchWarpZone` allows private instances by evaluating their base index, but `mto` calls coordinate-only `WarpSet` and never supplies the target private map index. Remote CCI map state is also stored from coordinate-derived base map index.

An allowed private target therefore routes the monarch to the base/public map, while the command still charges treasury and sets MI_WARP.

Promoted as `BUG-MON-011`.

## BUG-MON-012 — remote mtr loses the monarch's private-instance identity
Local `mtr` preserves `ch->GetMapIndex()`, including private instance IDs. The remote path instead sends `TPacketGGTransfer{name,x,y}`; receiver uses coordinate-only `WarpSet(x,y)`.

When the monarch is in an allowed private instance, the remote target is routed to the base/public map rather than the monarch's instance, while treasury and MI_TRANSFER are consumed.

Promoted as `BUG-MON-012`.
