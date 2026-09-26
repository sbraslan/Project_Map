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

Targeted source scan of `ranking_system.cpp`, `battle_field.cpp`, `input_p2p.cpp`, `p2p.cpp`, `input_db.cpp`, `db.cpp`, the descriptor/P2P helpers and packet declarations found the `TPacketGGLoadRanking` struct plus the P2P receiver, but no outbound construction/send site. The receive path is therefore currently mapped as **receiver-only / sender not found**. Cross-channel impact still needs reachability closure before assigning a bug ID.

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

Current mapped code does not show ranker-effect refresh being performed for already-online players when ranking data is reloaded. `SetWeakRankingPosition` sets `AFF_BATTLE_RANKER_1..3` directly through `CHARACTER::SetAffectFlag`, but does not clear an older ranker bit first.

Cross-checks show `m_afAffectFlag` is initialized to zero in the `CHARACTER` constructor, while the generic reset path in `char_affect.cpp` only resets `pkAff->dwFlag` for real `CAffect` entries. The mapped BattleField ranker bits are not added through that `CAffect` path. This makes stale winner flags a high-confidence lifecycle defect, but a repo-wide direct-reset scan is still required before promotion to a verified bug ID.

## Dynamic packet boundary audit
`HEADER_GC_BATTLE_ZONE_INFO` is registered as a dynamic-size packet. `CheckPacket()` waits until the declared dynamic size is buffered, but it does not validate that the declared size is at least the base packet size or that the payload length is an exact multiple of `sizeof(TBattleRankingMember)`.

`RecvBattleZoneInfo()` then subtracts `sizeof(TPacketGCBattleInfo)` from the `uint16_t wSize` and loops while `wSize > 0`, consuming one full ranking member each iteration. A too-small size can underflow; a non-multiple payload can cause a full member read beyond the packet's declared boundary. This creates BUG-RANK-005.

## BattleField ranking reload call-site audit
Both `CBattleField::CloseEnter()` and the scheduled weekly update call `LoadRanking(RK_CATEGORY_BF)` unqualified under `ENABLE_RANKING_SYSTEM`.

The current `CBattleField` declaration contains no `LoadRanking` member, its `singleton<CBattleField>` base contains no such method, and the directly included headers/precompiled-header chain checked so far contains no `LoadRanking` macro/alias. The actual method is `CRankingSystem::LoadRanking(uint8_t)`. The active `CommonDefines.h` enables both `ENABLE_RANKING_SYSTEM` and `ENABLE_BATTLE_FIELD`, so these call sites are compiled in the mapped configuration. This creates BUG-RANK-006 as a static build blocker for the source snapshot, subject only to an external compiler-injected declaration/macro not represented in the repository.

## Verified bugs
- BUG-RANK-001 — `SendBFRanking` takes `&vecBattleFieldRanking[0]` even when the vector is empty.
- BUG-RANK-002 — current-player solo ranking API is a hard-coded empty stub.
- BUG-RANK-003 — weekly winner table is not cleared, allowing stale prior-week positions.
- BUG-RANK-004 — BattleField close reloads ranking before remaining players' final session points are committed.
- BUG-RANK-005 — dynamic BattleField ranking packet length is not boundary/divisibility validated before fixed-record parsing.
- BUG-RANK-006 — BattleField calls `LoadRanking(RK_CATEGORY_BF)` without a resolvable member/global declaration in the active source configuration.

## Latent / incomplete integration findings
Not yet promoted to verified current-path bugs:
- Generic PARTY board calls Python APIs that are not exported by `PythonRankingModule.cpp`.
- Generic SOLO board declares categories beyond 0/1, while its name dictionary only defines 0 and 1.
- Ranker winner-effect state is not visibly refreshed for already-online players when ranking cache changes, and direct ranker bits are not cleared in the mapped lifecycle; repo-wide direct-reset closure is still pending.
- `TPacketGGLoadRanking` has a mapped receiver but no sender in the targeted server-source scan; cross-channel reachability/impact remains pending.

## Exact next audit
1. Map every live caller of `CBattleField::OpenBattleUI` and determine whether the missing outbound `TPacketGGLoadRanking` sender creates stale cross-channel ranking views.
2. Finish the repo-wide direct-reset search for `AFF_BATTLE_RANKER_1..3` and decide whether to promote the stale-ranker-effect lifecycle to a verified bug.
3. Determine whether any live caller opens generic PARTY ranking.
4. Audit remaining SQL/result null boundaries.
5. Reconcile BUG-RANK-006 against the actual build entry points/configuration and then decide whether Ranking is STATIC COMPLETE.


## Build/linkage anomaly under review
`battle_field.cpp` contains two unqualified calls to `LoadRanking(RK_CATEGORY_BF)`.

Current bounded checks found:
- no `CBattleField::LoadRanking` declaration in `battle_field.h`;
- no free `LoadRanking` declaration in the directly inspected BattleField includes;
- no `LoadRanking` macro in the game Makefile flags inspected;
- the actual implemented loader is `CRankingSystem::LoadRanking(uint8_t)`.

This is **not yet assigned a bug ID**. It may represent a missing wrapper/declaration or a compile-time integration defect, but the full transitive include graph has not yet been exhausted. Keep it as an explicit next-check item rather than assuming intended behavior.
