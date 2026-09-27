# World Lottery System

**Status:** STATIC COMPLETE
**Phase:** Detection / Mapping Only
**Closed:** 2026-09-27

> Source/game repositories are read-only. Only Project_Map is writable.

## Scope mapped
Server / DB:
- `game/src/char_lottery.cpp`
- `game/src/input_main.cpp::LottoOpenWindow/LottoBuyTicket/LottoTicketOptions/LottoPickMoney`
- `game/src/packet.h` and `packet_info.cpp`
- `game/src/char.cpp::PointChange/CreatePlayerProto/SendPoints/save_event`
- `game/src/char.h` lottery accessors
- `db/src/LottoManager.cpp/.h`
- `db/src/ClientManager.cpp` 10-second scheduler
- `common/CommonDefines.h`
- `common/tables.h::TPlayerTable`
- `libsql/AsyncSQL.cpp/.h`

Client:
- `UserInterface/PythonNetworkStreamModule.cpp`
- `UserInterface/PythonNetworkStreamPhaseGame.cpp`
- `UserInterface/PythonNetworkStreamPhaseLoading.cpp`
- `UserInterface/Packet.h`
- `UserInterface/PythonPlayer.cpp/.h`
- `UserInterface/PythonPlayerModule.cpp`
- `root/game.py`
- `root/constinfo.py`
- `root/interfacemodule.py`
- `root/uilottery.py`
- `root/uiscript/lottery*.py`

## Core flow
Official UI buys one of three visible tickets with four selected numbers.
Python -> C++ network binding -> fixed CG packet -> `CInputMain` -> SQL ticket row.

DB core polls every 10 seconds. Draw history is stored in `lotto_numbers`; ticket results are written back into `lotto_tickets`, results are inserted into `log.lotto_log`, and the next draw row is inserted.

Claim changes a ticket to collected and credits two player lottery points:
- current lottery wallet;
- lifetime lottery winnings.

Withdrawal transfers lottery wallet value into normal gold.

## Configuration
- `ENABLE_WORLD_LOTTERY_SYSTEM`: enabled.
- `NEW_LOTTERY_NUMBERS_ACTIVATE = 1`.
- Ticket cost: 5,000,000 Yang.
- 75% of ticket cost is added to the next jackpot.
- Initial jackpot path: 100,000,000.
- Minimum next jackpot: 250,000,000.
- Configured generation interval: 2 minutes.
- Recurring implementation currently hardcodes 30 seconds (BUG-WLOT-010).

## Client/cache notes
- Ticket cache is explicitly reset when the server sends `tID = 0`.
- Base-number history is written by numeric slot without clearing higher stale slots; this matters if DB history shrinks/resets, but no ordinary append-only runtime trigger was found.
- Ranking client dictionaries are append-only and do not overwrite/clear existing entries.
- Ranking transport exists, but the shipped lottery UI has no ranking caller/button. Ranking defects are therefore marked dormant for the official UI while remaining reachable through the existing binding/handler.

## DB/async notes
- Draw-result UPDATE/log/next-row statements use the same async SQL queue and are consumed FIFO. No independent async reordering bug was found.
- Character names are validated through `check_name_alphabet` for Turkey/Germany and accept only letters/digits, so the currently interpolated normal character name does not expose a verified quote-based SQL injection path.
- The `next_time - 10` threshold is checked by a 10-second scheduler with strict `<`; under nominal cadence it does not independently prove an early-draw defect. No bug ID assigned.

## Verified bugs
- `BUG-WLOT-001` — arbitrary server ticket slots bypass the intended three-ticket limit.
- `BUG-WLOT-002` — duplicate ticket numbers can turn one actual match into a 4/4 jackpot result.
- `BUG-WLOT-003` — persisted 64-bit lottery values cross 32-bit PointChange, full-points packet, and client-status boundaries.
- `BUG-WLOT-004` — withdrawal debits lottery wallet before final gold-cap success is known.
- `BUG-WLOT-005` — out-of-range stored ticket numbers can index outside the official client's 1..30 ticket grid.
- `BUG-WLOT-006` — negative withdrawal amounts can decrease gold while increasing lottery wallet.
- `BUG-WLOT-007` — `COUNT(*)` is treated as current/future draw identity; row/id gaps can stall or desynchronize draws.
- `BUG-WLOT-008` — result log writes the previous draw id rather than the evaluated draw id.
- `BUG-WLOT-009` — every 4/4 winner independently receives the full jackpot, over-allocating multi-winner draws.
- `BUG-WLOT-010` — recurring draw schedule hardcodes 30 seconds instead of the configured two minutes.
- `BUG-WLOT-011` — dormant jackpot ranking sends `lotto_ticket_id` as packet/client `lottoID`.
- `BUG-WLOT-012` — dormant money ranking can null-dereference a missing `player_index` empire row.
- `BUG-WLOT-013` — claim commits ticket state before winnings are durably saved, creating a crash-window permanent prize loss.
- `BUG-WLOT-014` — unchecked lottery `DirectQuery` errors can feed null result pointers into `mysql_fetch_row`.

## Deferred runtime validation
See `tests/world_lottery.md`. Runtime/fault-injection work remains disabled in the current project phase.

## Closure
Static source/client/DB mapping for World Lottery is complete at the recorded source snapshots.
No source code, Python client code, quests, or game data were modified.
