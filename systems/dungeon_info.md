# dungeon info

**Status:** STATIC COMPLETE

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


## Active checkpoint — Dungeon Info static close audit started

**Tarih:** 2026-09-26

Ticket System STATIC COMPLETE sonrasında Dungeon Info aktif subsystem oldu.

### Already verified
- core server/client/network flow mapped;
- BUG-DUNGEON-001..006 recorded.

### Exact next audit
1. validate CG/GC packet-info size/sequence and all dungeon index boundaries;
2. audit config parser invariants and fixed-array capacities;
3. audit ranking query/result lifecycle and Python binding boundaries;
4. close client clear/reload/state lifecycle;
5. decide Dungeon Info STATIC COMPLETE and refresh tests.

## Static close — Dungeon Info

**Tarih:** 2026-09-26

### Packet / sequence audit — COMPLETE
- CG DungeonInfo is registered server-side with exact `sizeof(TPacketCGDungeonInfo)` and sequence=true.
- Official client `SendDungeonInfo` sends the fixed struct and calls `SendSequence()`.
- GC DungeonInfo and Ranking are registered client-side as STATIC packets with their exact struct sizes.
- No header-size/sequence mismatch was found.

### Current config snapshot audit — COMPLETE
`Project_Game/share/locale/europe/dungeon_info.txt` currently contains 9 dungeon blocks.
- required-item count: exactly 3 per block (packet capacity 3)
- boss-drop maximum: 5 (packet capacity 16)
- bonus maximum: 7 (below POINT_MAX_NUM)
- each block has one LEVEL_LIMIT and one ENTRY_BASE_POSITION
- QUEST-backed blocks: 4
- explicit COOLDOWN lines: 0

Thus BUG-DUNGEON-004/005/006/007 are real defensive/config-parser defects but are not triggered by current vector/token sizes. BUG-DUNGEON-010/011/012 do affect the normal checked-in configuration/path.

### Client state / Python boundary audit — COMPLETE
- `Clear()` only clears slot 0 -> BUG-DUNGEON-003.
- 255-slot storage accepts index 255 / narrowing boundaries -> BUG-DUNGEON-002.
- nested bonus/item getter indices are unvalidated -> BUG-DUNGEON-008.
- `TPacketGCDungeonInfo` itself has a constructor that zero-initializes scalar/fixed-array fields; an earlier uninitialized-packet suspicion was rejected as a false positive.

### Ranking audit — COMPLETE
- server Warp/Ranking primary index validation is missing -> BUG-DUNGEON-001.
- Ranking SQL is malformed at the adjacent literal boundary before LEFT JOIN -> BUG-DUNGEON-009.
- ranking result packet has safe default initialization; terminator row is ignored by AddRanking because level=0 while still causing UI refresh.
- no additional confirmed MYSQL_ROW null dereference was found in the normal ranking loop.

### Config semantics / cooldown audit — COMPLETE
- unbounded token `strcpy` -> BUG-DUNGEON-007.
- documented numeric GLOBAL flag contract disagrees with parser -> BUG-DUNGEON-011.
- expired/zero cooldown subtraction wraps through uint32 -> BUG-DUNGEON-012.

### UI audit — COMPLETE
- CORRECTION 2026-09-29: the `for key in xrange(...)` list-button loop is aligned after the `if/else`, not nested inside the zero-count branch. `BUG-DUNGEON-010` is retracted.

## Static status
Dungeon Info: **STATIC COMPLETE**.

Verified bugs: `BUG-DUNGEON-001..009`, `BUG-DUNGEON-011`, `BUG-DUNGEON-012`. `BUG-DUNGEON-010` retracted.
Runtime/ASan validation remains in `../tests/dungeon_info.md`.

Next project phase: runtime / in-game bug validation.


## Correction checkpoint — 2026-09-29

A fresh preflight against the current tracked client snapshot disproved the old UI-control-flow finding.

Current `root/uidungeoninfo.py::DungeonInfoWindow.Initialize`:
- line 697 checks `dungeonInfo.GetCount() > 0`;
- the zero-count `else` ends before the list construction loop;
- line 709 `for key in xrange(min(...))` is at the function-body indentation level and therefore executes for nonzero counts.

The checked-in `dungeon_info.txt` still contains 9 blocks, so normal OPEN should create list buttons.

Decision:
- `BUG-DUNGEON-010` = **RETRACTED / FALSE POSITIVE**;
- `DUNGEON-T10` = **RETRACTED / DO NOT RUN**;
- previous dependency claim that T09/T11/T12 were blocked by T10 is removed;
- first future live gate moves to `DUNGEON-T09`.

The SQL defect behind `BUG-DUNGEON-009` was reverified in current `game/src/DungeonInfo.cpp`: the query literal ends with `...dungeon_ranking`` and the immediately adjacent next literal starts `LEFT JOIN...` with no separating whitespace.
