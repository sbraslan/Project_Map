# Monarch — Static Map

**Status:** STATIC MAPPING IN PROGRESS / 4 VERIFIED BUGS  
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

## BUG-MON-003 — rmmonarch calls deletion twice and cannot publish a successful removal
Registered GM command:
`rmmonarch <name>`
→ `HEADER_GD_RMMONARCH`
→ DB `CClientManager::RMMonarch`.

The handler executes:
1. `CMonarch::Instance().DelMonarch(szName);`
2. immediately calls the same function again to compute `iRet`.

If the first call succeeds, it removes/clears the matching monarch, so the second call cannot find the same name and returns false. The success broadcast branch therefore cannot represent the successful first mutation.

There is also no `HEADER_DG_UPDATE_MONARCH_INFO` refresh in this path.

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

## Open candidates
- multi-core treasury double-spend/effect-before-authoritative-deduction;
- process-local PowerUp/DefenseUp effects versus multi-core empire scope;
- `takemonarchmoney` has authorization/precheck code under `__UNIMPLEMENTED__`; audit tracked caller reachability;
- admin `SetMonarch` SQL schema/column consistency;
- `DelMonarch` DELETE result handling;
- warp/transfer money/cooldown committed without checking final warp/transfer success;
- election vote/candidacy producer deployment is incomplete in tracked Game quest corpus.

## Current audit cursor
1. close treasury request/ack concurrency and failure semantics;
2. map `takemonarchmoney` callers and authorization;
3. map PowerUp/DefenseUp deployed callers and cross-core behavior;
4. audit monarch warp/transfer failure charging;
5. audit Set/Del SQL persistence details;
6. decide additional promotions.

## Runtime
No Monarch runtime/fault-injection test may execute while the global execution lock is active. First future live gate remains `DUNGEON-T09`.
