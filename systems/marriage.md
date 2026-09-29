# Marriage / Wedding — Static Map

**Status:** STATIC COMPLETE / 8 VERIFIED BUGS  
**Phase:** Detection / Mapping Only  
**Opened:** 2026-09-29  
**Source/Game repositories:** READ-ONLY  
**Runtime:** LOCKED / NOT RUN

## Scope
Engagement creation, marriage persistence, wedding private-map orchestration, engaged-to-married transition, login/setup reconstruction, wedding termination and divorce/removal.

## Verified source roots
- `Project_ServerSRC/game/src/marriage.cpp`
- `Project_ServerSRC/game/src/marriage.h`
- `Project_ServerSRC/game/src/questlua_marriage.cpp`
- `Project_ServerSRC/game/src/wedding.cpp`
- `Project_ServerSRC/game/src/wedding.h`
- `Project_ServerSRC/game/src/input_db.cpp`
- `Project_ServerSRC/game/src/questlua_pc.cpp`
- `Project_ServerSRC/db/src/Marriage.cpp`
- `Project_ServerSRC/db/src/Marriage.h`
- `Project_ServerSRC/db/src/ClientManager.cpp`
- `Project_ServerSRC/common/tables.h`
- `Project_Game/share/locale/europe/quest/n_npc/marriage_manage.quest`
- `Project_Game/share/locale/europe/quest/quest_list`
- `Project_Game/share/locale/europe/map/metin2_map_wedding_01/*`
- deployed channel/core CONFIG files.

## Deployment
`MAP_WEDDING_01 = 81`.

Current tracked CONFIG ownership:
- `Project_Game/chan/ch2/core4/CONFIG` contains map 81.
- `Project_Game/chan/ch99/core99/CONFIG` also contains map 81.

The deployed quest list contains `n_npc/marriage_manage.quest`.

## Verified initial flows

### 1. Engagement creation
`marriage_manage.quest`
→ level/ring/sex/empire/dress/proximity checks
→ `confirm(u_vid,...)`
→ initiator pays 1,000,000 Yang
→ both 70301 engagement rings are converted to 70302
→ quest displays text and suspends at `wait()`
→ `marriage.engage_to(u_vid)`
→ `ALUA(marriage_engage_to)`
→ `marriage::CManager::RequestAdd`
→ `HEADER_GD_MARRIAGE_ADD`
→ DB `CClientManager::MarriageAdd`
→ DB `marriage::CManager::Add`
→ SQL `INSERT INTO marriage`
→ `HEADER_DG_MARRIAGE_ADD`
→ game `CInputDB::MarriageAdd`
→ game `marriage::CManager::Add`.

When both partners are online at game-side Add, the game core also emits `HEADER_GD_WEDDING_REQUEST`.

### BUG-MARR-001 — engagement resources commit before authoritative marriage creation
The deployed quest commits Yang and both ring conversions before the authoritative relation request, and then explicitly suspends at `wait()`.

If the quest is interrupted before it resumes, or if the target is no longer resolvable by the stored VID when `marriage.engage_to(u_vid)` finally runs, the Lua binding returns without `CManager::RequestAdd`. No rollback restores the already committed resources.

Promoted as `BUG-MARR-001`.

### 2. Wedding map orchestration
game `HEADER_GD_WEDDING_REQUEST`
→ DB `CClientManager::WeddingRequest`
→ unfiltered `ForwardPacket(HEADER_DG_WEDDING_REQUEST,...)`
→ all connected game peers
→ `CInputDB::WeddingRequest`
→ `WeddingManager::Request`
→ every receiving core for which `map_allow_find(81)` is true can create a private wedding map
→ loads `metin2_map_wedding_01/npc.txt`
→ `HEADER_GD_WEDDING_READY`
→ DB `CClientManager::WeddingReady`
→ `HEADER_DG_WEDDING_READY` + DB `ReadyWedding(...)`
→ 5-second DB start queue
→ `HEADER_DG_WEDDING_START`
→ game `CInputDB::WeddingStart`
→ `marriage::CManager::WeddingStart`
→ partner warp through the currently stored `pWeddingInfo->dwMapIndex`.

DB tracks running weddings and schedules the end boundary using `WEDDING_LENGTH = 60 * 60` seconds.

### BUG-MARR-002 — deployed map 81 ownership allows duplicate wedding-map producers
DB `WeddingRequest` broadcasts to every game peer with no channel filter. Current deployment has map 81 enabled on both ch2/core4 and ch99/core99.

Therefore both cores can independently execute `WeddingManager::Request`, create a private wedding map and send `HEADER_GD_WEDDING_READY` for the same pair.

DB `ReadyWedding` queues each READY without pair-level deduplication. Game `WeddingReady` retains only one `pWeddingInfo->dwMapIndex`, so later READY delivery can overwrite the previous map index. `TPacketWeddingStart` carries only PID1/PID2, not the selected map index.

Promoted as `BUG-MARR-002`.

### 3. Engaged → married transition
`marriage_manage.quest`
→ `marriage.set_to_marriage()`
→ `ALUA(marriage_set_to_marriage)`
→ `TMarriage::SetMarried`
→ `TMarriage::Save`
→ `CManager::RequestUpdate`
→ `HEADER_GD_MARRIAGE_UPDATE`
→ DB `marriage::CManager::Update`
→ SQL update of `love_point/is_married`
→ `HEADER_DG_MARRIAGE_UPDATE`
→ game `CInputDB::MarriageUpdate`.

### 4. Divorce / relation removal
`marriage_manage.quest`
→ `marriage.remove()`
→ `ALUA(marriage_remove)`
→ game `CManager::RequestRemove`
→ `HEADER_GD_MARRIAGE_REMOVE`
→ DB `marriage::CManager::Remove`
→ SQL `DELETE FROM marriage`
→ `HEADER_DG_MARRIAGE_REMOVE`
→ game `CInputDB::MarriageRemove`
→ game relation removal.

### 5. Game-core reconstruction
DB `marriage::CManager::OnSetup` replays persisted relations to a game peer using:
- `HEADER_DG_MARRIAGE_ADD`
- `HEADER_DG_MARRIAGE_UPDATE`
- running wedding `HEADER_DG_WEDDING_READY`
- running wedding `HEADER_DG_WEDDING_START`.

## DB packet map
- GD 70 / DG 150 — MARRIAGE_ADD
- GD 71 / DG 151 — MARRIAGE_UPDATE
- GD 72 / DG 152 — MARRIAGE_REMOVE
- GD 73 / DG 153 — WEDDING_REQUEST
- GD 74 / DG 154 — WEDDING_READY
- DG 155 — WEDDING_START
- GD 75 / DG 156 — WEDDING_END
- GD 116 / DG 159 — BREAK_MARRIAGE legacy headers. DB GD receiver parses two PIDs and routes into normal marriage::CManager::Remove; no independent modern divorce flow was established.

## Additional verified findings

### BUG-MARR-003 — mutual divorce rejects exactly 500,000 Yang
The deployed mutual-divorce quest defines `MONEY_NEED_FOR_ONE = 500000` but computes both players' eligibility with strict `> MONEY_NEED_FOR_ONE` checks, both before and after confirmation. A character with exactly 500,000 Yang is therefore treated as insufficient even though the subsequent debit is exactly 500,000.

Promoted as `BUG-MARR-003`.

### BUG-MARR-004 — stale P2P logout can overwrite fresh lover-online state
Cross-core P2P login updates an existing name-keyed CCI to the newest descriptor. P2P logout is later processed only by player name and does not verify that the logout came from the descriptor currently stored in that CCI.

The already-verified channel/core ordering race therefore also reaches Marriage:
`LOGIN(new core)`
→ existing CCI descriptor updated
→ Marriage/login path can emit `lover_login`
→ delayed `LOGOUT(old core)`
→ name-only CCI removal
→ `marriage::CManager::Logout(pid)`
→ `TMarriage::Logout`
→ `lover_logout` sent to spouse/current relay.

The client handles `lover_logout` by marking the lover offline and hiding lover state. No later login event is guaranteed because the fresh LOGIN already happened before the stale logout.

Promoted as `BUG-MARR-004`.

### BUG-MARR-005 — wedding exit uses the wrong saved-location fields
`SaveExitLocation()` stores:
- `m_posExit = GetXYZ()`
- `m_lExitMapIndex = GetMapIndex()`.

But `ExitToSavedLocation()` calls:
`WarpSet(m_posWarp.x, m_posWarp.y, m_lWarpMapIndex)`
instead of using `m_posExit / m_lExitMapIndex`, then clears the real exit fields.

Wedding entry explicitly calls `SaveExitLocation()`, and wedding shutdown calls `ExitToSavedLocation()` for every PC. After a completed warp, `WarpEnd()` resets `m_posWarp` and `m_lWarpMapIndex` to zero, so the wedding exit path does not use the saved pre-wedding destination.

Promoted as `BUG-MARR-005`.

### BUG-MARR-006 — same-process warp does not detach WeddingMap membership
`CHARACTER::SetWeddingMap(nullptr)` is the code path that calls `WeddingMap::DecMember`, but `CHARACTER::WarpSet()` does not call it.

Wedding shutdown:
1. `SetEnded()` schedules the end event;
2. step 0 calls `WeddingMap::WarpAll()`;
3. `WarpAll()` calls `ExitToSavedLocation()/WarpSet()`;
4. 15 seconds later the same event calls `WeddingManager::DestroyWeddingMap()`;
5. `DestroyWeddingMap()` calls `WeddingMap::DestroyAll()`, which destroys every character still in `m_set_pkChr`.

If the exit warp stays in the same game process/core, character destruction does not occur during the warp, so the stale WeddingMap member entry survives and that already-warped player is destroyed/disconnected 15 seconds later.

Current deployment makes same-core exits realistic because map 81 shares cores with other allowed maps (for example ch2/core4 also hosts 61..70 and ch99/core99 hosts multiple normal/special maps).

Promoted as `BUG-MARR-006`.

### BUG-MARR-007 — EXP-based love-point progression becomes zero from level 26 onward
EXP distribution computes:
`static_cast<uint32_t>(2000.0L / level / level / 3) * static_cast<uint32_t>(iFinalExp)`.

The fractional level coefficient is cast to `uint32_t` before multiplication. Marriage requires level 25 in the deployed quest:
- level 25 coefficient is about 1.066 and truncates to 1;
- level 26 coefficient is below 1 and truncates to 0;
- every higher level remains 0.

Therefore the entire EXP-derived update is zero for level 26+ regardless of EXP earned. The subsequent `pMarriage->Update(dwUpdatePoint)` receives zero and does not set/save love-point progress.

Promoted as `BUG-MARR-007`.

### BUG-MARR-008 — marriage item sharing breaks when spouses are on different cores
The tracked locale item descriptions for 71069..71074 explicitly state that when one spouse wears the item, the effect applies to **both spouses**.

Server implementation:
- `GetMarriageBonus` delegates to `TMarriage::GetBonus(..., bShare=true)`.
- Shared bonus checks only local `ch1/ch2` pointers and their equipped items.
- Those pointers are populated by `marriage::CManager::Login(ch)` for characters local to that game process.
- P2P login updates CCI/guild/party/messenger state but does not populate remote Marriage character pointers.

Therefore when spouse A and spouse B are online on different game cores:
- A's core can see A's equipped marriage item but not B;
- B's core can see B but not A;
- if only A wears the item, B's `GetMarriageBonus` returns no shared effect.

This contradicts the deployed item description's both-spouses behavior and makes the bonus dependent on process placement rather than marriage/equipment state.

Promoted as `BUG-MARR-008`.

## Closed / scoped observations
- Legacy `HEADER_GD_BREAK_MARRIAGE` is a DB-side two-PID compatibility entry that calls the normal DB marriage remove routine. No separate active quest/client sender was established in the tracked deployed flow; `HEADER_DG_BREAK_MARRIAGE` has no independent Marriage gameplay consequence in the mapped path.
- Marriage critical/penetration/EXP bonus consumers are server-side. No separate item-bonus defect is promoted in this pass.
- Mutual-divorce stale-target/double-mutation candidate is closed for the normal confirmation path: `CQuestManager::Confirm` resumes the suspended quest synchronously inside the confirmation handler, and the post-confirm mutual-divorce branch contains no additional suspension before its target re-check/mutations. This differs from BUG-MARR-001, where a later explicit `wait()` creates a real interruption window.

## Static closure
Marriage / Wedding static mapping is complete for the tracked source/deployment snapshot.

Verified bugs:
- `BUG-MARR-001..008`.

Closed/scoped:
- legacy BREAK_MARRIAGE is a compatibility DB remove entry, not an independent active gameplay flow;
- mutual-divorce stale-target double-mutation is not reachable in the ordinary confirm path because confirm resumes synchronously and no later suspension occurs;
- deployed Marriage Lua calls guard the obvious null-state paths through quest predicates; no additional independent Lua crash was promoted;
- post-insert WeddingMap GetMap failure cleanup remains low-probability/deferred without a demonstrated ordinary producer;
- active-wedding relation removal/orphaning remains unpromoted without an ordinary deployed path that reaches outside divorce NPCs during the active private-map window;
- marriage item consumers for attack/defense/transfer/critical/penetration/EXP are present; the cross-core sharing defect is owned by BUG-MARR-008.

## Runtime
No Marriage runtime test may be executed while the global execution lock is active. First future live gate remains `DUNGEON-T09`.


### BUG-MARR-009 — DB startup deletes every pending engagement from persistent storage
DB startup calls `MarriageManager.Initialize()`. Before loading relations it executes:
`DELETE FROM marriage WHERE is_married = 0`.

Engagements are represented by the same marriage table with `is_married = 0`, so every pending engagement is permanently removed on each DB server restart.

The deployed quest already contains a cleanup branch allowing a non-married player holding 70302 to discard the ring, but it does not reconstruct the relation or refund the engagement cost/resources.

Promoted as `BUG-MARR-009`.

### BUG-MARR-010 — Marriage Fast makes computed love points non-monotonic
`GetMarriagePoint()` does not persist daily love-point accrual. It recomputes the entire time component from `get_global_time() - marry_time` using the **current** premium state:
- normal: 1 point/day, max 30;
- Marriage Fast active while both spouse pointers are local: 2 points/day, max 40.

Therefore enabling Marriage Fast retroactively applies the doubled daily rate to all elapsed marriage days, while expiry (or loss of same-process `IsOnline()`) recomputes the same history at the slower rate and can reduce the displayed/effective marriage point.

Example with zero stored kill points after 20 marriage days:
- premium active: `50 + min(20*2,40) = 90`;
- premium inactive: `50 + min(20,30) = 70`.

A “points increase twice as fast” effect therefore can cause already-visible love points to drop later.

Promoted as `BUG-MARR-010`.
