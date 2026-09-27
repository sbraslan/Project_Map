# World Lottery System

**Status:** PARTIAL — ACTIVE
**Phase:** Detection / Mapping Only
**Date:** 2026-09-27

> Source/game repositories are read-only. Only Project_Map is writable.

## Server roots
- `game/src/char_lottery.cpp`
- `game/src/input_main.cpp::LottoOpenWindow/LottoBuyTicket/LottoTicketOptions/LottoPickMoney`
- `game/src/packet.h` lottery packet structs
- `game/src/packet_info.cpp` CG lottery registrations
- `game/src/char.cpp::PointChange` lottery/gold point handling
- `game/src/char.h` lottery wallet accessors
- `db/src/LottoManager.cpp/.h`
- `db/src/ClientManager.cpp` 10-second lottery scheduler
- `common/CommonDefines.h` lottery constants
- `common/tables.h::TPlayerTable` lottery persistence fields

## Client roots
- `root/uilottery.py`
- `root/uiscript/lottery*.py`
- `UserInterface/PythonNetworkStreamModule.cpp` lottery Python bindings
- `UserInterface/PythonNetworkStream*.cpp` lottery send/receive/header paths
- `UserInterface/Packet.h` lottery packet structs

## Feature/config state
- `ENABLE_WORLD_LOTTERY_SYSTEM` enabled.
- New draws enabled: `NEW_LOTTERY_NUMBERS_ACTIVATE = 1`.
- Ticket cost: 5,000,000 Yang.
- 75% of ticket cost is added to next jackpot.
- Initial jackpot: 100,000,000 in DB manager first-row path.
- Minimum next jackpot: 250,000,000.

## Core data model
Player table persists:
- `lotto_moneypool` as `long long`
- `lotto_totalmoneywin` as `long long`.

Ticket DB row stores:
- player id/name
- arbitrary integer slot
- four integer selected numbers
- target draw id
- state
- matched count
- money win.

## Ticket purchase flow
Python:
`AddTicketWindow.BuyNewTicket`
-> `m2netm2g.LottoBuyTicket(slot,n1,n2,n3,n4)`
-> fixed CG packet
-> `CInputMain::LottoBuyTicket`.

Server currently checks only:
- whether the exact player+slot already exists;
- whether player has ticket cost.

It does not validate:
- slot is 1..3;
- numbers are 1..30;
- four numbers are distinct.

This creates BUG-WLOT-001 and BUG-WLOT-002.

## Draw scheduler
DB core calls:
`CLottoManager::CheckRefreshTime()`
every 10 seconds.

First draw row:
- generates 4 unique random numbers in 1..30;
- inserts next draw row.

Refresh path:
1. reads current row count and current draw timing;
2. when refresh threshold is reached, generates 4 unique numbers;
3. selects tickets for the next draw id;
4. compares each of the four stored ticket fields independently against the four winning numbers;
5. computes payout from matched count;
6. updates ticket state/money;
7. logs result;
8. inserts next draw row.

Because each stored ticket field is counted independently, duplicate client numbers can inflate `win_numbers`; see BUG-WLOT-002.

## Prize claim flow
`SUBHEADER_CG_RECIVE_MONEY`:
- loads ticket by player+slot;
- validates state;
- changes ticket state to collected;
- calls `PointChange(POINT_LOTTO_MONEY, atoll(money_win))`;
- calls `PointChange(POINT_LOTTO_TOTAL_MONEY, atoll(money_win))`.

Lottery storage fields are `long long`, but `PointChange` accepts an `int amount`.
Large prizes are therefore narrowed before being added; see BUG-WLOT-003.

## Lottery wallet withdrawal
`LottoPickMoney` accepts signed `long long amount`.

Flow:
1. checks only current gold < GOLD_MAX;
2. checks lottery wallet >= requested amount;
3. subtracts amount from lottery wallet;
4. adds amount to gold.

The wallet is changed before gold-cap success is known. If gold addition is rejected by the normal GOLD_MAX guard, wallet deduction remains. See BUG-WLOT-004.

The same flow also passes `long long` through `PointChange(int)`, so large withdrawal amounts share BUG-WLOT-003's narrowing boundary.

## Packet framing
CG lottery packets are registered with their exact fixed struct sizes and sequence=true:
- OPENINGS
- BUY_TICKET
- TICKET_OPTIONS
- PICK_MONEY.

Python bindings accept raw integers/long-long and do not add security validation; server-side validation is therefore required.

## Client receive/cache closure
- Base-info packets are received into `constInfo.lotto_number_infos[lottoSlot]`. The server numbers each transmitted row from slot 0 upward and sends at most the latest 50 rows. Existing higher cache slots are not cleared before a new snapshot; normal append-only history stays coherent, but DB row deletion/reset can leave stale higher entries. This remains a mapping note until a normal runtime trigger is found.
- Ticket slots are cleaner: the server always sends slots 1..3, including an explicit `tID = 0` packet for an empty slot, and `game.py` resets every cached ticket field when that packet arrives.
- Ranking packets are received into two append-only dictionaries. Existing jackpot entries are suppressed by `lottoID`; money-ranking entries are suppressed by `playername`. Existing values are never overwritten and neither dictionary is cleared before a fresh server top-10 snapshot, so a repeated ranking request can retain stale values and stale entries.
- The ranking transport is present (`LottoOpenRanking` Python binding -> C++ send -> server `SendLottoRankingInfo`), but no caller/button or ranking window was found in the shipped `uilottery.py` / `LotteryMainWindow.py`. For now this is recorded as a dormant client defect, not promoted to a verified user-facing bug.
- Ticket numbers are copied from server cache directly into a fixed 1..30 UI grid without bounds checks. Because the server accepts arbitrary ticket numbers, a stored out-of-range number can raise a Python index error on ticket refresh; see BUG-WLOT-005.

## Negative withdrawal boundary
`TPacketCGSendLottoPickMoney.amount` is signed `long long`, but the handler does not require `amount > 0`.

For a simple negative request such as `-100`:
- `GetLottoMoney() >= amount` is true for a normal non-negative wallet;
- `PointChange(POINT_LOTTO_MONEY, -amount)` adds 100 to the lottery wallet;
- `PointChange(POINT_GOLD, amount)` subtracts 100 gold.

The gold branch has no lower-bound guard and `SetGold(int)` accepts negative values. Thus the withdrawal endpoint can create negative gold while increasing the lottery wallet. Larger negative long-long values also cross the existing long-long -> int narrowing boundary and can produce implementation-dependent sign/value inversions. See BUG-WLOT-006.

## Verified bugs
- `BUG-WLOT-001` — no server slot bound; arbitrary slot values bypass the intended three-ticket limit.
- `BUG-WLOT-002` — duplicate ticket numbers can turn one winning number into a 4/4 jackpot match.
- `BUG-WLOT-003` — lottery balances/prizes are long long but PointChange takes int, causing narrowing/corruption for large values.
- `BUG-WLOT-004` — wallet is deducted before gold overflow rejection, causing deterministic withdrawal loss.
- `BUG-WLOT-005` — out-of-range stored ticket numbers are used as unchecked 1..30 UI-grid indexes and can break the official lottery client refresh.
- `BUG-WLOT-006` — negative withdrawal amounts are accepted and can drive gold negative while increasing lottery wallet; large negatives also interact with the narrowing boundary.

## Draw-id / payout accounting audit
The DB manager derives the current draw identity from:
`SELECT COUNT(*) FROM player.lotto_numbers`

and then uses that count as if it were the current `lotto_id`:
`WHERE lotto_id = COUNT(*)`.

Ticket purchase also assigns:
`for_lotto_id = COUNT(*) + 1`.

This requires the table to remain forever contiguous from id 1 with no deleted/missing row and no auto-increment gap. A single gap breaks that invariant: the scheduler can query a nonexistent current id and return without producing a new draw, while ticket targets can be assigned to the wrong future id. Ticket jackpot contribution already uses `MAX(lotto_id)`, which confirms the codebase itself mixes two incompatible notions of "current draw". See BUG-WLOT-007.

When a draw is evaluated, tickets selected are:
`for_lotto_id = COUNT(*) + 1`.

The generated winning numbers are then inserted as the next draw row. However the result log writes:
`lotto_id = COUNT(*)`.

Under the normal contiguous case this records the previous draw id rather than the draw the ticket was evaluated against. See BUG-WLOT-008.

Each 4/4 ticket receives:
`win_money = jackpot`.

There is no winner count and no split operation. With two legitimate jackpot winners, the system promises two full jackpots and subtracts `2 * jackpot` from `next_jackpot`; with N winners it subtracts `N * jackpot`. The later minimum floor hides the negative/overdrawn intermediate accounting rather than sharing the available jackpot. See BUG-WLOT-009.

The first draw schedules the next generation with `60 * 2` seconds, matching `GENERATE_NEW_LOTTO_NUMBERS_PULSE_MIN = 2`. The recurring path explicitly comments out the configured calculation and hardcodes `next_refresh = 30`. After the first draw, the configured two-minute interval is therefore ignored and draws are scheduled every 30 seconds. See BUG-WLOT-010.

## Ranking safety / semantic mapping
`SendLottoRankingInfo()` queries jackpot rows as:
`player_name, lotto_ticket_id, money_win, date`

and assigns `lotto_ticket_id` to a packet field named `lottoID`. Thus if the dormant ranking transport is used, the client cache key/display value called `lottoID` is actually a ticket id, not the logged draw id. This is recorded as BUG-WLOT-011; the shipped lottery UI currently has no ranking caller.

For total-money ranking, each player account performs:
`SELECT empire FROM player.player_index WHERE id = account_id`.

The code checks only SQL error, not whether a row exists. It then unconditionally evaluates `atoi(row_empire[0])`. A missing `player_index` row therefore dereferences a null MYSQL_ROW and can crash the game process when ranking is requested. See BUG-WLOT-012. The official UI path remains dormant, but the server handler and Python binding are present.

## Verified bugs
- `BUG-WLOT-001` — no server slot bound; arbitrary slot values bypass the intended three-ticket limit.
- `BUG-WLOT-002` — duplicate ticket numbers can turn one winning number into a 4/4 jackpot match.
- `BUG-WLOT-003` — lottery balances/prizes are long long but PointChange takes int, causing narrowing/corruption for large values.
- `BUG-WLOT-004` — wallet is deducted before gold overflow rejection, causing deterministic withdrawal loss.
- `BUG-WLOT-005` — out-of-range stored ticket numbers are used as unchecked 1..30 UI-grid indexes and can break the official lottery client refresh.
- `BUG-WLOT-006` — negative withdrawal amounts are accepted and can drive gold negative while increasing lottery wallet.
- `BUG-WLOT-007` — COUNT(*) is treated as the current draw id; any row/id gap can stall or desynchronize draw scheduling and ticket targeting.
- `BUG-WLOT-008` — result logging writes the previous draw id instead of the evaluated next draw id.
- `BUG-WLOT-009` — every 4/4 winner receives the full jackpot independently, so multiple jackpot winners overdraw the pot instead of sharing it.
- `BUG-WLOT-010` — recurring draws hardcode 30 seconds and ignore the configured two-minute generation interval.
- `BUG-WLOT-011` — jackpot ranking labels/keys `lotto_ticket_id` as `lottoID`, so ranking draw identity is semantically wrong when used.
- `BUG-WLOT-012` — ranking empire lookup dereferences a missing player_index row without a row-count/null check.

## Next audit
1. Finish ticket delete/claim SQL-error/null handling and mutation ordering.
2. Audit async DB update ordering around draw result rows versus next-draw insertion.
3. Audit claim/withdrawal persistence and save ordering.
4. Check lottery point serialization widths from server player points through client storage.
5. Reconcile all World Lottery findings and decide STATIC COMPLETE readiness.
