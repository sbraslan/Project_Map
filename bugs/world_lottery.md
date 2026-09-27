# World Lottery System — Bug Registry

**Phase:** Detection / Mapping Only

### BUG-WLOT-001 — server does not enforce the three valid lottery ticket slots
- Statik durum: **doğrulandı**
- Sınıf: client trust / ticket-limit bypass

`TPacketCGSendLottoNewTicket.slot` is a client-controlled `int`.

`CInputMain::LottoBuyTicket` checks only whether the exact `player_id + slot` pair already has a ticket. It never validates that slot is one of the UI slots 1, 2 or 3.

A modified client can therefore buy tickets in arbitrary slots (4, 5, 100, negative values, etc.). Each distinct slot passes the existing-ticket check independently.

The intended UI limit is three active ticket slots, but the server does not enforce it; a player can create more than three tickets for the same draw as long as ticket cost is available.

### BUG-WLOT-002 — duplicate selected numbers can be counted as four matches and trigger jackpot payout
- Statik durum: **doğrulandı**
- Sınıf: jackpot integrity / client input validation

The official UI selects four distinct numbers in range 1..30, but the server does not validate either uniqueness or range.

The DB draw evaluator checks each stored ticket field independently:
- number1 matches any winning number -> +1
- number2 matches any winning number -> +1
- number3 matches any winning number -> +1
- number4 matches any winning number -> +1.

Therefore a crafted ticket such as:
`7, 7, 7, 7`

is recorded unchanged.

If 7 appears once among the four legitimate draw numbers, all four ticket fields independently match that one number:
`win_numbers == 4`

and the code executes:
`win_money = jackpot`.

This allows one actual matched draw number to be interpreted as a 4/4 jackpot match.

### BUG-WLOT-003 — long long lottery values are narrowed through PointChange(int)
- Statik durum: **doğrulandı**
- Sınıf: numeric truncation / currency corruption

Lottery storage and protocol amounts use `long long`:
- `lotto_moneypool`
- `lotto_totalmoneywin`
- ticket `money_win`
- pick-money packet `amount`.

However:
`CHARACTER::PointChange(uint16_t type, int amount, ...)`
accepts only a 32-bit signed `int`.

Prize claim passes:
`atoll(row[2])`
into `PointChange(POINT_LOTTO_MONEY, ...)` and `POINT_LOTTO_TOTAL_MONEY`.

Wallet withdrawal also passes signed `long long amount` through the same int parameter.

Any lottery value outside the signed-int range is narrowed before the point mutation. The jackpot has no matching int-range cap and can grow through ticket contributions, so the type mismatch is reachable by system design.

Consequences include wrong-sign/wrong-value wallet and lifetime-win updates while the ticket can already be marked collected.

### BUG-WLOT-004 — lottery wallet is deducted before gold-cap rejection
- Statik durum: **doğrulandı**
- Sınıf: transaction ordering / deterministic currency loss

`CInputMain::LottoPickMoney` checks:
- current gold is below GOLD_MAX;
- wallet contains the requested amount.

It does **not** check:
`current_gold + requested_amount < GOLD_MAX`.

It then performs:
1. `PointChange(POINT_LOTTO_MONEY, -amount)`
2. `PointChange(POINT_GOLD, amount)`.

The gold handler independently rejects additions whose resulting gold reaches/exceeds GOLD_MAX.

Therefore a request can pass the initial checks, successfully deduct the lottery wallet, and then have the gold addition rejected.

Result: the withdrawn amount disappears from the lottery wallet without being added to inventory gold.

This is separate from BUG-WLOT-003 and occurs even with ordinary int-range amounts.


### BUG-WLOT-005 — out-of-range ticket numbers can break the official client lottery refresh
- Statik durum: **doğrulandı**
- Sınıf: server input validation / persistent client-side failure

The server accepts the four client-controlled ticket numbers without enforcing the intended range 1..30.

Those values are stored unchanged and later returned by `SendLottoTicketInfo()`.

The official client builds each ticket grid as indices 1..30 and then executes the equivalent of:
`NumberGrids[row][ticket_number].SetCheckNumber(True)`

without validating the received ticket number.

Therefore a crafted stored value such as 100 is outside the 31-entry grid and raises a Python indexing error when that ticket is refreshed. The malformed value persists in SQL, so the failure survives reopening/relogin until the ticket row is removed or corrected.

This is a separate consequence from BUG-WLOT-002: duplicate values corrupt jackpot matching, while out-of-range values can poison the official client's own lottery refresh path.

### BUG-WLOT-006 — negative withdrawal amounts can create invalid negative gold and inflate the lottery wallet
- Statik durum: **doğrulandı**
- Sınıf: signed-input validation / currency invariant violation

`TPacketCGSendLottoPickMoney.amount` is a client-controlled signed `long long`.

`CInputMain::LottoPickMoney` does not require the amount to be positive. Its wallet check is only:
`GetLottoMoney() >= amount`

For a normal non-negative wallet, any modest negative amount passes.

Example with `amount = -100`:
1. `PointChange(POINT_LOTTO_MONEY, -amount)` adds 100 to the lottery wallet.
2. `PointChange(POINT_GOLD, amount)` subtracts 100 from gold.

The `POINT_GOLD` branch checks only the upper GOLD_MAX boundary. It has no lower-bound rejection, and `SetGold(int)` stores the resulting negative value directly.

A client can therefore use the withdrawal endpoint in the opposite direction and push character gold below zero while increasing the lottery wallet. This violates both currency invariants and endpoint semantics.

Because the packet is long long while `PointChange` accepts int, very large negative values also cross BUG-WLOT-003's narrowing boundary and may invert/truncate the effective mutations depending on the target compiler conversion behavior.


### BUG-WLOT-007 — COUNT(*) is treated as the current draw id
- Statik durum: **doğrulandı**
- Sınıf: draw identity / database continuity invariant

Both the scheduler and ticket-purchase path derive draw identity from `COUNT(*)`.

The scheduler queries the supposed current row with:
`WHERE lotto_id = COUNT(*)`

and future tickets are assigned to:
`COUNT(*) + 1`.

This is valid only if every historical id from 1 through the maximum id exists forever. If any row is removed or an auto-increment gap exists, row count and maximum/current id diverge.

A missing current-count id makes the scheduler's info query return zero rows and the function returns, so new draws can stop. Even when a row happens to exist, `COUNT+1` can identify the wrong target draw.

The purchase path simultaneously updates jackpot contribution using `MAX(lotto_id)`, proving the implementation mixes MAX-id and row-count semantics.

### BUG-WLOT-008 — result log stores the previous draw id
- Statik durum: **doğrulandı**
- Sınıf: audit-log integrity / off-by-one draw mapping

During refresh, the evaluator selects tickets with:
`for_lotto_id = COUNT(*) + 1`.

Those tickets are evaluated against the newly generated numbers that are inserted as the next draw row.

But each result log writes:
`lotto_id = COUNT(*)`.

In the normal contiguous case the ticket belongs to draw N+1 while the log records draw N. Jackpot/history records are therefore attached to the previous draw id.

### BUG-WLOT-009 — multiple jackpot winners each receive the entire jackpot
- Statik durum: **doğrulandı**
- Sınıf: jackpot accounting / over-allocation

For every ticket with four matches the evaluator independently sets:
`win_money = jackpot`.

There is no pre-count of jackpot winners and no division of the pot.

Two 4/4 tickets therefore create `2 * jackpot` in promised prizes; N winners create `N * jackpot`.

All payouts are accumulated into `new_jackpot_wins` and subtracted from `next_jackpot`. The subsequent minimum-jackpot floor can mask a negative/overdrawn intermediate balance but does not make the original allocation conserved.

### BUG-WLOT-010 — recurring draw interval ignores configured generation period
- Statik durum: **doğrulandı**
- Sınıf: scheduling/configuration drift

`GENERATE_NEW_LOTTO_NUMBERS_PULSE_MIN` is configured as 2 minutes.

The initial-row path schedules 120 seconds, consistent with that setting.

The recurring path contains the configured calculation only as a comment and instead executes:
`int next_refresh = 30;`

Thus after initialization, draws are scheduled every 30 seconds instead of the configured two minutes.

### BUG-WLOT-011 — jackpot ranking sends ticket id as lottoID
- Statik durum: **doğrulandı — dormant official UI path**
- Sınıf: ranking semantic mapping

The jackpot ranking query selects:
`player_name, lotto_ticket_id, money_win, date`.

The second column is assigned to:
`TPacketGCSendRankingJackpotInfo.lottoID`.

The client then stores/deduplicates that value as `lottoID`.

So the transport labels a ticket id as a draw/lottery id. This corrupts ranking draw identity whenever the ranking endpoint is invoked. The shipped lottery UI currently contains no ranking caller, although the Python binding and server handler exist.

### BUG-WLOT-012 — missing player_index row can crash lottery ranking request
- Statik durum: **doğrulandı — dormant official UI path**
- Sınıf: SQL result safety / null dereference

For each total-money ranking row, the server queries:
`SELECT empire FROM player.player_index WHERE id = account_id`.

It checks only `uiSQLErrno`.

Then:
`MYSQL_ROW row_empire = mysql_fetch_row(...)`
is followed unconditionally by:
`atoi(row_empire[0])`.

If the query succeeds but returns zero rows, `row_empire` is null and the code dereferences it. A ranking request can therefore crash the game process for inconsistent/orphaned player data.

The current shipped lottery UI does not expose the ranking request, so this path is dormant for normal UI use, but it is reachable through the existing network handler/binding.
