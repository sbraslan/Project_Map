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

- WLOT-T13 claim a winning ticket, confirm ticket state becomes 2, then terminate the game process before the next character save; after restart compare ticket state against persisted `lotto_moneypool` / `lotto_totalmoneywin`. Covers BUG-WLOT-013.
- WLOT-T14 in a controlled test environment, force a lottery DirectQuery failure before its unchecked `mysql_fetch_row` path and verify null-result handling/crash behavior. Covers BUG-WLOT-014.
- WLOT-T15 persist lottery wallet/total-win above INT_MAX, relog, and compare DB `long long` values with the full `TPacketGCPoints`/client status values. Extends BUG-WLOT-003 coverage.


## World Lottery readiness consolidation — 2026-09-28
- No World Lottery runtime test was executed.
- WLOT-T01 -> BUG-WLOT-001.
- WLOT-T02 -> BUG-WLOT-002.
- WLOT-T03 + WLOT-T15 -> BUG-WLOT-003.
- WLOT-T04 -> BUG-WLOT-004.
- WLOT-T05 -> BUG-WLOT-005.
- WLOT-T06 -> BUG-WLOT-006.
- WLOT-T07 -> BUG-WLOT-007.
- WLOT-T08 -> BUG-WLOT-008.
- WLOT-T09 -> BUG-WLOT-009.
- WLOT-T10 -> BUG-WLOT-010.
- WLOT-T11 -> BUG-WLOT-011.
- WLOT-T12 -> BUG-WLOT-012.
- WLOT-T13 -> BUG-WLOT-013.
- WLOT-T14 -> BUG-WLOT-014.
- Modified-client/adversarial: WLOT-T01, T02, T05, T06.
- High-value/numeric boundary: WLOT-T03, T04, T15.
- DB/state-shaping: WLOT-T07, T08, T09, T10.
- Dormant ranking endpoint: WLOT-T11, T12.
- Crash/fault-injection: WLOT-T13, T14.
- Primary ordinary/current-flow candidates: WLOT-T04 and WLOT-T10; WLOT-T03/T15 require high-value boundary state.
- Overall first live runtime gate remains DUNGEON-T09.
