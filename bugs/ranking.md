# Ranking System — Bug Registry

**Phase:** Detection / Mapping Only

### BUG-RANK-001 — empty BattleField ranking vector is indexed with operator[]
- Statik durum: **doğrulandı**
- Sınıf: C++ undefined behavior / empty container access

`CRankingSystem::SendBFRanking` builds a dynamic BattleField ranking packet and then sends:

`&vecBattleFieldRanking[0]`

with length:
`sizeof(TBattleRankingMember) * vecBattleFieldRanking.size()`.

If the vector is empty, `operator[](0)` is invalid even though the byte count passed to `Packet` is zero.

Normal reachability exists:
- `CBattleField::OpenBattleUI` calls `SendBFRanking`;
- ranking vectors can be empty when the corresponding DB queries return no rows.

No source change is authorized in the current phase.

### BUG-RANK-002 — current player's BattleField ranking row can never be populated
- Statik durum: **doğrulandı**
- Sınıf: incomplete client binding / deterministic UI defect

`PythonRankingModule.cpp::rankingGetMyRankingInfoSolo` is a hard-coded stub:

it always returns ranking index 0, empty name, zero records/time/empire.

The active `root/uibattlefield.py::RefreshRankingList` calls this function for the current-player row in both weekly and accumulated ranking views.

Result:
- top-10 ranking rows can be shown;
- the dedicated current-player ranking row is always empty, regardless of the player's actual ranking.

### BUG-RANK-003 — weekly winner rollover leaves stale previous-week positions
- Statik durum: **doğrulandı**
- Sınıf: DB lifecycle / stale ranking state

`CBattleField::UpdateWeekRanking` selects at most three players from the current week's positive `week_score` rows and REPLACEs only positions 1..N into `log.battle_week`.

It never deletes/truncates positions that are not overwritten.

If a later week has fewer than three qualifying players:
- old rows for remaining positions stay in `log.battle_week`;
- `CRankingSystem::LoadRankingWeekWinners` reads that table ordered by score and LIMIT 3;
- prior-week players can therefore remain in the active winner vector.

If a week has zero qualifying players, no positions are overwritten at all before week scores are reset.

### BUG-RANK-004 — BattleField close reloads ranking before final player scores are persisted
- Statik durum: **doğrulandı**
- Sınıf: ordering / stale cache

`CBattleField::CloseEnter` calls:

`LoadRanking(RK_CATEGORY_BF)`

before iterating the BattleField map and calling `ExitCharacter` for remaining players.

`ExitCharacter` is the path that calls `RegisterBattleRanking` and commits each player's temporary session points into `log.battle_score`.

Therefore the cache reload performed at close is built from DB state that does not yet contain the final points of players still inside the map.

The immediate post-close ranking cache can be stale until a later reload occurs.

### BUG-RANK-005 — malformed BattleField ranking packet size can escape the declared packet boundary
- Statik durum: **doğrulandı**
- Sınıf: client packet parser / dynamic-size boundary validation

`HEADER_GC_BATTLE_ZONE_INFO` is registered as a dynamic packet. `CheckPacket()` only waits until the packet's declared `uint16_t size` is buffered; it does not validate a minimum base size or payload divisibility.

`RecvBattleZoneInfo()` then:
- receives `TPacketGCBattleInfo`;
- subtracts `sizeof(TPacketGCBattleInfo)` from `wSize`;
- while `wSize > 0`, receives a full `TBattleRankingMember` and subtracts that full struct size.

Therefore:
- a declared size smaller than the base packet can underflow the unsigned size field;
- a payload whose length is not an exact multiple of `sizeof(TBattleRankingMember)` can make the parser request bytes beyond the declared packet boundary and desynchronize/fail the receive stream.

This requires a malformed server packet; no client-to-server exploit path is asserted here.

### BUG-RANK-006 — RETRACTED: suspected unresolved `LoadRanking` call
- Statik durum: **geri çekildi / yanlış pozitif**
- Sınıf: audit correction

The original finding missed a transitive include path.

Verified resolution:
- `cmd_general.cpp` defines global `LoadRanking(uint8_t)` and sends `TPacketGGLoadRanking` through `P2P_MANAGER::Send` before reloading the local `CRankingSystem` cache;
- `cmd.h` declares the helper;
- `battle_field.cpp -> char.h -> horse_rider.h -> cmd.h` makes that declaration visible to the BattleField translation unit.

Therefore the two unqualified `LoadRanking(RK_CATEGORY_BF)` calls are resolvable and this is not a build blocker.

The ID remains reserved as a retracted finding and is not reused.

### BUG-RANK-007 — online BattleField winner flags are not refreshed when ranking changes
- Statik durum: **doğrulandı**
- Sınıf: ranking lifecycle / stale direct affect flags

`CBattleField::Connect` calls `SetWeakRankingPosition`, which looks up the current top-3 winner vector and directly sets one of:
- `AFF_BATTLE_RANKER_1`
- `AFF_BATTLE_RANKER_2`
- `AFF_BATTLE_RANKER_3`

The ranking reload paths are:
- local global helper `LoadRanking(uint8_t)`;
- P2P `CInputP2P::LoadRanking`;
- `CRankingSystem::LoadRanking/LoadBFRanking`.

Those paths reload ranking vectors only. They do not iterate online characters and do not call `SetWeakRankingPosition`.

In addition, `SetWeakRankingPosition` does not clear an older ranker bit before setting a new one.

Consequences at weekly rollover while players remain online:
- an old top-3 player can keep the previous winner flag after losing the position;
- an already-online new top-3 player does not receive the corresponding flag from the ranking reload itself;
- reconnect/recreation can rebuild the flag state, but the live reload path does not.

## Findings awaiting reachability closure

### Generic PARTY board API gap
Client build enables `ENABLE_RANKING_SYSTEM_PARTY` and constructs `uiRankingBoard.RankingBoardWindow`.

That UI calls:
- `ranking.GetHighRankingInfoParty`
- `ranking.GetMyRankingInfoParty`
- party-member helper APIs.

Those methods are not exported by the active `PythonRankingModule.cpp`; they only appear inside a commented-out method block.

No verified bug ID is assigned yet because no active server/client caller for `TYPE_RK_PARTY` has been mapped.

### Generic SOLO category dictionary gap
The generic ranking module exposes solo categories through index 7, but `SOLO_RANK_BOARD_NAME` in `uirankingboard.py` currently contains only keys 0 and 1.

Opening a generic solo category 2..7 would index a missing dictionary key.

No verified bug ID is assigned until an active caller for those categories is mapped.

### Ranker-effect refresh gap
Promoted to **BUG-RANK-007** after reload and online-character lifecycle closure.

### P2P ranking reload sender gap — CLOSED
The sender exists in `cmd_general.cpp::LoadRanking(uint8_t)`.

It constructs `TPacketGGLoadRanking`, broadcasts through `P2P_MANAGER::Send`, then reloads the local ranking cache. The earlier “sender not found” note was incomplete and is closed.
