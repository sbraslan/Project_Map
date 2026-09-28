# Ranking System — Deferred Test Notes

**Status:** DEFERRED — documentation only
**Current phase:** Detection / Mapping Only

Do not execute runtime/fault-injection tests unless the user explicitly changes project phase.

- RANK-T01: empty `log.battle_score`, open BattleField UI under debug/ASan; covers BUG-RANK-001.
- RANK-T02: place current character inside/outside top 10 and inspect the dedicated “my ranking” row; covers BUG-RANK-002.
- RANK-T03: seed `battle_week` with three old winners, perform rollover with one new qualifying player, inspect stale positions; covers BUG-RANK-003.
- RANK-T04: keep scored players inside BattleField until forced close; compare cache immediately after close with DB after `ExitCharacter`; covers BUG-RANK-004.
- RANK-T05: if a live PARTY ranking caller is later found, open it and verify whether missing Python APIs cause an AttributeError.
- RANK-T06: after winner rollover while winners are online, inspect old/new ranker effects; reserved for lifecycle closure.


- RANK-T07: feed an isolated/debug client a malformed BATTLE_ZONE_INFO packet with (a) declared size smaller than the base header and (b) payload length not divisible by TBattleRankingMember; inspect parser boundary handling/desynchronization. Covers BUG-RANK-005. This is a malformed-server-packet parser test, not a client-to-server exploit test.

## Ranking readiness consolidation — 2026-09-28
- No Ranking runtime test was executed.
- RANK-T01 -> BUG-RANK-001.
- RANK-T02 -> BUG-RANK-002.
- RANK-T03 -> BUG-RANK-003.
- RANK-T04 -> BUG-RANK-004.
- RANK-T06 -> BUG-RANK-007.
- RANK-T07 added for BUG-RANK-005.
- BUG-RANK-006 remains RETRACTED / RESERVED and has no runtime validation target.
- RANK-T05 remains conditional integration coverage for the dormant generic PARTY API gap; it is not mapped to a verified bug.
- The generic SOLO category 2..7 dictionary gap remains documented-only because no active caller was found.
- Primary normal-path candidates: RANK-T01, RANK-T02, RANK-T03 and RANK-T04.
- RANK-T07 is isolated client-parser robustness testing.
- Overall first live runtime gate remains DUNGEON-T10.
