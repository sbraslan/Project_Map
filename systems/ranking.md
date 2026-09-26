# Ranking System

**Status:** PARTIAL — ACTIVE
**Phase:** Detection / Mapping Only
**Date:** 2026-09-26

> Source repositories are strictly read-only. This file is the canonical static map for the active Ranking subsystem.

## Scope
This subsystem currently covers the generic Ranking layer and its active BattleField integration.

### Server roots
- `game/src/ranking_system.h`
- `game/src/ranking_system.cpp`
- `game/src/battle_field.cpp`
- `game/src/battle_field.h`
- `game/src/input_p2p.cpp`
- `game/src/packet.h`
- `game/src/packet_info.cpp`

### Client C++ roots
- `UserInterface/PythonRanking.h`
- `UserInterface/PythonRanking.cpp`
- `UserInterface/PythonRankingModule.cpp`
- `UserInterface/Packet.h`
- `UserInterface/PythonNetworkStream.cpp`
- `UserInterface/PythonNetworkStreamPhaseGame.cpp::RecvBattleZoneInfo`

### Client Python/UI roots
- `root/uibattlefield.py` — active BattleField ranking UI
- `root/uirankingboard.py` — generic ranking board
- `root/interfacemodule.py::OpenRankingBoardWindow`
- `root/game.py::OpenRankingBoard`

## Feature state
Active source configuration:
- server: `ENABLE_RANKING_SYSTEM`
- server: `ENABLE_BATTLE_FIELD`
- client: `ENABLE_RANKING_SYSTEM`
- client: `ENABLE_RANKING_SYSTEM_PARTY`
- client: `ENABLE_BATTLE_FIELD`

## Server ranking model
`CRankingSystem` currently implements BattleField ranking category `RK_CATEGORY_BF = 0`.

Solo categories declared:
- 0: BattleField weekly
- 1: BattleField total
- 2: MD red
- 3: MD blue
- 6: Black & White
- 7: World Boss

Only BattleField loading is implemented in the mapped server ranking manager.

### Load lifecycle
`CRankingSystem::LoadBFRanking()`:
1. clears `vecBattleFieldRanking`;
2. loads weekly top 10;
3. loads total top 10;
4. reloads weekly winner IDs.

Weekly list query reads `log.battle_score.week_score`.
Total list query reads `total_score + week_score`.
Winner-effect list reads `log.battle_week`, ordered by score, top 3.

## BattleField score persistence
`CBattleField::RegisterBattleRanking`:
- checks whether a `log.battle_score` row exists for the player;
- UPDATEs `week_score += session points`, or REPLACEs a new row;
- updates `last_update`.

`CBattleField::ExitCharacter` commits the player's temporary BattleField points through this function before warping the character out.

## Weekly rollover
`CBattleField::UpdateWeekRanking()`:
1. selects the current top 3 from `log.battle_score.week_score`;
2. REPLACEs positions 1..N into `log.battle_week`;
3. moves all week_score values into total_score;
4. resets week_score to zero.

The table is not cleared before positions are rewritten. This creates BUG-RANK-003.

## Ranking reload triggers
Mapped triggers:
- BattleField scheduled ranking-update time: `UpdateWeekRanking()` then `LoadRanking(RK_CATEGORY_BF)`.
- BattleField close: `LoadRanking(RK_CATEGORY_BF)`.
- P2P receive: `HEADER_GG_LOAD_RANKING -> CInputP2P::LoadRanking -> CRankingSystem::LoadRanking(category)`.

`HEADER_GG_LOAD_RANKING` is registered with exact `sizeof(TPacketGGLoadRanking)`.

The exact outbound/broadcast helper that emits `TPacketGGLoadRanking` still needs to be located.

## GC packet flow
Server BattleField UI open:
`CBattleField::OpenBattleUI`
-> `CRankingSystem::SendBFRanking`
-> `HEADER_GC_BATTLE_ZONE_INFO`.

Packet:
- base: `TPacketGCBattleInfo { header, wSize }`
- payload: zero or more `TBattleRankingMember`.

Server member layout:
- position
- category
- name
- empire
- score.

Client layout uses the name `bType` instead of `bCategory` for the same second byte. The binary layout matches; this naming difference is **not a bug**.

Client header map registers `HEADER_GC_BATTLE_ZONE_INFO` as dynamic with base size `sizeof(TPacketGCBattleInfo)`.

`RecvBattleZoneInfo()`:
1. receives base packet;
2. clears cached BattleField ranking;
3. subtracts base size;
4. receives fixed `TBattleRankingMember` records;
5. stores them in `CPythonRanking`;
6. calls Python `OpenRankingBoard(0, 0)`.

## Active UI path
`root/interfacemodule.py::OpenRankingBoardWindow(type, category)` routes:
- `type == 0 && category == 0` to `wndBattleField.Open()`;
- other categories/types may route to the generic ranking board.

Therefore the normal BattleField ranking UI is `root/uibattlefield.py`, not the generic `uirankingboard.py`.

BattleField UI:
- CURRENT_RANK -> weekly category;
- ACCUM_RANK -> total category;
- both read top rows through `ranking.GetHighRankingInfoSolo`;
- current-player row uses `ranking.GetMyRankingInfoSolo`.

## Python ranking module
Currently exported:
- `GetHighRankingInfoSolo`
- `GetMyRankingInfoSolo`.

`GetHighRankingInfoSolo` scans the cached BattleField vector by category and position.

`GetMyRankingInfoSolo` is currently a hard-coded empty stub and never derives the current player's position/data. This creates BUG-RANK-002.

Party-ranking functions are present only in a commented method block and are not exported.

## Ranker winner effects
`CBattleField::Connect` calls `SetWeakRankingPosition`.

That function:
- looks up the player in the cached top-3 winner vector;
- sets one of `AFF_BATTLE_RANKER_1..3` when matched.

Current mapped code does not show ranker-effect refresh being performed for already-online players when ranking data is reloaded. It also does not clear previous ranker flags inside `SetWeakRankingPosition`.

This is recorded as a lifecycle finding; persistence/reachability must be closed before assigning another verified bug ID.

## Verified bugs
- BUG-RANK-001 — `SendBFRanking` takes `&vecBattleFieldRanking[0]` even when the vector is empty.
- BUG-RANK-002 — current-player solo ranking API is a hard-coded empty stub.
- BUG-RANK-003 — weekly winner table is not cleared, allowing stale prior-week positions.
- BUG-RANK-004 — BattleField close reloads ranking before remaining players' final session points are committed.

## Latent / incomplete integration findings
Not yet promoted to verified current-path bugs:
- Generic PARTY board calls Python APIs that are not exported by `PythonRankingModule.cpp`.
- Generic SOLO board declares categories beyond 0/1, while its name dictionary only defines 0 and 1.
- Ranker winner-effect state is not visibly refreshed for already-online players when ranking cache changes.
- Exact outbound P2P ranking-reload sender still needs mapping.

## Exact next audit
1. Locate the outbound `TPacketGGLoadRanking` sender/broadcast path.
2. Close ranker-effect refresh/removal lifecycle.
3. Determine whether any live caller opens generic PARTY ranking.
4. Audit dynamic ranking packet length handling for malformed/non-multiple payload sizes.
5. Audit SQL/result null handling and then decide whether Ranking is STATIC COMPLETE.
