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

- WLOT-T07 create an id gap in `lotto_numbers` while preserving rows; run refresh and verify `COUNT(*)` no longer identifies the active max id and scheduling/targeting stalls or diverges. Covers BUG-WLOT-007.
- WLOT-T08 on a contiguous history, evaluate tickets for draw N+1 and verify `log.lotto_log.lotto_id` is written as N. Covers BUG-WLOT-008.
- WLOT-T09 create two legitimate 4/4 winners in one draw; verify both receive the full jackpot and `new_jackpot_wins` becomes twice the pot. Covers BUG-WLOT-009.
- WLOT-T10 compare the configured 2-minute generation constant against recurring inserted `next_numbers`; verify recurring rows use 30 seconds. Covers BUG-WLOT-010.
- WLOT-T11 invoke ranking data and compare packet/client `lottoID` against `lotto_log.lotto_ticket_id` and `lotto_id`. Covers BUG-WLOT-011.
- WLOT-T12 with an otherwise valid total-money ranking player whose `player_index` row is absent, invoke ranking and observe the null-row dereference path. Covers BUG-WLOT-012. Do not execute until runtime-testing phase is explicitly enabled.
