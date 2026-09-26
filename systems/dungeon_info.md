# dungeon info

**Status:** STATIC MAPPED

> Canonical subsystem history split from legacy `00_PROGRESS.md`. Read this file only when this subsystem is active or explicitly revisited.

## Checkpoint — legacy Ticket / Dungeon Info / Battle Pass audit reconciled

**Tarih:** 2026-09-26

Mailbox STATIC COMPLETE sonrasında geçmiş sohbetlerde incelenmiş fakat Project_Map'e yazılmamış üç subsystem kaynak kodla tekrar doğrulandı ve canonical haritaya geri alındı.

### Ticket System
Kaynak doğrulaması:
- Server: `game/src/ticket.cpp` SHA `6f63e12241d21f0be7d7649767398e7d28045a66`
- Server input: `game/src/input_main.cpp::CInputMain::TicketSystem`
- Packet: `HEADER_CG_TICKET_SYSTEM=129`, `HEADER_GC_TICKET_SYSTEM=148`
- Client: `UserInterface/PythonTicket.cpp`
- Binary UI: `root/uiticket.py`

Akış:
`uiticket.py -> PythonTicket binding -> CG Ticket packet -> CInputMain::TicketSystem -> CTicketSystem::{Open,Create,Reply,Action,ChangePage} -> ticket.list/reply/user_restricted -> GC Ticket packet/client cache`.

Doğrulanan buglar:
- BUG-TICKET-001 reply-page ownership bypass / foreign ticket replies readable by crafted ID.
- BUG-TICKET-002 raw SQL string interpolation + blacklist escaping weakness.
- BUG-TICKET-003 ticket-ID collision loop stale query result nedeniyle infinite loop/string growth.
- BUG-TICKET-004 arbitrary admin sort mode -> uninitialized SQL query buffer; paging LIMIT count also cumulative.
- BUG-TICKET-005 client Request(id) boundary check `size() < id` nedeniyle id==size OOB.

### Dungeon Info
Kaynak doğrulaması:
- Server: `game/src/DungeonInfo.cpp` SHA `61ad7b1177772249da6d4d0c28286f21cb3c0de7`
- Packet/input: `game/src/packet.h`, `input_main.cpp::DungeonInfo`
- Client: `UserInterface/PythonDungeonInfo.cpp/.h`
- Network: `PythonNetworkStreamPhaseGame.cpp`

Akış:
`uidungeoninfo.py/dungeonInfo module -> CPythonDungeonInfo -> SendDungeonInfo -> CG 159 -> CInputMain::DungeonInfo -> CDungeonInfoManager::{SendInfo,Warp,Ranking} -> GC dungeon packets -> CPythonDungeonInfo/UI`.

Doğrulanan buglar:
- BUG-DUNGEON-001 server Warp/Ranking unchecked dungeon index.
- BUG-DUNGEON-002 client 255-element array accepts uint8 index 255 -> OOB.
- BUG-DUNGEON-003 CPythonDungeonInfo::Clear only clears slot 0, stale slots survive reload/clear.
- BUG-DUNGEON-004 Warp iterates level-limit count while indexing entry-position vector.
- BUG-DUNGEON-005 config vectors required-item/boss-drop are copied into fixed packet arrays without cap.
- BUG-DUNGEON-006 bonus cap check uses `iAffect > POINT_MAX_NUM`, permitting index == POINT_MAX_NUM.

### Battle Pass
Kaynak doğrulaması:
- Server manager: `game/src/battle_pass.cpp` SHA `fb4d2f2cc96fd1c2db8e61fe7a26322c3f434ca6`
- Character progress: `game/src/char.cpp` SHA `78e0917ec13cddd1db7f2287da50aa007f7b85f0`
- Event integration: `game/src/event_manager.cpp`
- Client receive: `PythonNetworkStreamPhaseGame.cpp::RecvExtBattlePassMissionUpdatePacket`
- Binary callbacks: `root/game.py -> root/uibattlepass.py`

Akış:
game event/activity
-> `CHARACTER::UpdateExtBattlePassMissionProgress` / `SetExtBattlePassMissionProgress`
-> in-memory mission state
-> mission reward
-> `TPacketGCExtBattlePassMissionUpdate`
-> client callback/UI
-> dirty mission persistence on character save/logout.

Doğrulanan ilk buglar:
- BUG-BPASS-001 mission-update packet leaves `bMissionType` uninitialized while client consumes it.
- BUG-BPASS-002 SetExtBattlePassMissionProgress resets completed mission to incomplete before re-evaluation, allowing repeat mission reward if helper is called again above threshold.
- BUG-BPASS-003 BattlePassRequestOpen stores `c_str()` from a block-local std::string and uses dangling pointer.
- BUG-BPASS-004 season name copied with unbounded `strcpy` into fixed packet buffer.
- BUG-BPASS-005 final reward SELECT does not validate zero-row/null MYSQL_ROW before row[0].
- BUG-BPASS-006 Event Manager cache arrays are not initialized by constructor; InitializeBattlePass calls CheckBattlePassTimes and can consume indeterminate values.
- BUG-BPASS-007 Event Manager `BattlePassData(..., bool bState)` writes bool 0/1 into the active-pass-ID array, so configured season IDs other than 1 are lost.

### Canonical durum
- Ticket System: recovered static audit; core path mapped, runtime tests remain.
- Dungeon Info: recovered static audit; core path mapped, runtime/ASan tests remain.
- Battle Pass: recovered audit is **PARTIAL**. Caller audit + persistence/season lifecycle still açık.

### Sıradaki
1. Battle Pass `SetExtBattlePassMissionProgress` caller audit.
2. Battle Pass mission create/load/save + `battlepass_playerindex` lifecycle.
3. Event/P2P season start-stop/reload state.
4. Bunlar kapandıktan sonra Battle Pass STATIC COMPLETE kararı.

## Related
- Bugs: `../bugs/dungeon_info.md`
- Runtime tests: `../tests/dungeon_info.md`
- Full legacy archive: `../archive/00_PROGRESS.md`
