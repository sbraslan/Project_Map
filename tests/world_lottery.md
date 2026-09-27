# World Lottery System — Deferred Test Notes

**Status:** DEFERRED — documentation only
**Current phase:** Detection / Mapping Only

Do not execute runtime/fault-injection tests unless the user explicitly changes project phase.

- WLOT-T01 send BUY_TICKET with slots 4, 100 and -1; verify additional rows can be created beyond UI slots. Covers BUG-WLOT-001.
- WLOT-T02 buy crafted ticket [7,7,7,7], force/observe draw containing 7 once, verify win_numbers becomes 4 and jackpot path is selected. Covers BUG-WLOT-002.
- WLOT-T03 use prize/wallet values above INT_MAX and compare long-long DB/protocol values with PointChange result. Covers BUG-WLOT-003.
- WLOT-T04 set gold near GOLD_MAX, withdraw an amount that passes current-gold check but would overflow final gold; verify wallet decreases while gold addition is rejected. Covers BUG-WLOT-004.

- WLOT-T05 create a crafted stored ticket containing an out-of-range number such as 100, then trigger `SendLottoTicketInfo`; verify official client refresh indexes outside the 1..30 grid. Covers BUG-WLOT-005.
- WLOT-T06 send `LottoPickMoney(-100)` with ordinary non-negative balances; verify lottery wallet increases by 100 and gold decreases below zero when insufficient. Separately probe large negative long-long values around 32-bit narrowing boundaries. Covers BUG-WLOT-006 and its interaction with BUG-WLOT-003.
