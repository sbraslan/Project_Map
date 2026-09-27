# World Lottery System — Deferred Test Notes

**Status:** DEFERRED — documentation only
**Current phase:** Detection / Mapping Only

Do not execute runtime/fault-injection tests unless the user explicitly changes project phase.

- WLOT-T01 send BUY_TICKET with slots 4, 100 and -1; verify additional rows can be created beyond UI slots. Covers BUG-WLOT-001.
- WLOT-T02 buy crafted ticket [7,7,7,7], force/observe draw containing 7 once, verify win_numbers becomes 4 and jackpot path is selected. Covers BUG-WLOT-002.
- WLOT-T03 use prize/wallet values above INT_MAX and compare long-long DB/protocol values with PointChange result. Covers BUG-WLOT-003.
- WLOT-T04 set gold near GOLD_MAX, withdraw an amount that passes current-gold check but would overflow final gold; verify wallet decreases while gold addition is rejected. Covers BUG-WLOT-004.
