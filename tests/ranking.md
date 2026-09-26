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
