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
`SetWeakRankingPosition` sets a winner effect when a player connects, but mapped ranking reload paths do not iterate online players to refresh winner effects.

The function itself also does not clear previous ranker flags.

The online-state update gap is structurally visible, but exact old-flag lifetime/reconnect behavior still needs mapping before a verified bug ID is assigned.
