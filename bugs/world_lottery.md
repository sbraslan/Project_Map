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
