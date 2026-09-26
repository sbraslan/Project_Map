# hunting

**Status:** PARTIAL — ACTIVE

> Canonical subsystem history split from legacy `00_PROGRESS.md`. Read this file only when this subsystem is active or explicitly revisited.

## Checkpoint — Hunting System audit started

**Tarih:** 2026-09-26

Biolog STATIC COMPLETE sonrasında Hunting System açıldı.

### Roots mapped
- Server runtime: `game/src/char_hunting.cpp`
- CG action dispatch: `input_main.cpp::ReciveHuntingAction`
- kill progress hook: `char_battle.cpp::UpdateHuntingMission`
- login/level-up bootstrap: `input_login.cpp::CheckHunting` + `char.cpp::CheckHunting`
- client UI: `root/uihunting.py`
- client send: `m2netm2g.SendHuntingAction`
- state/persistence: `hunting_system.*` quest flags.

### Action protocol
- action 1: open current/main window
- action 2: select mission type using client-controlled `dValue`
- action 3: open cached reward window
- action 4: receive cached rewards.

Official UI sends mission type 0/1.

### State machine
First `CheckHunting`:
- is_active 0 -> -1
- level -> 1
- type -> -1
- count -> 0.

Selection:
is_active -1 + player level >= hunting level
-> is_active 1
-> type = client dValue
-> count 0
-> main window.

Kill:
`char_battle -> UpdateHuntingMission(monsterVnum)`
-> target/count table match
-> increment quest flag
-> completion -> `SetCachedRewards`.

Cached reward flags:
- reward_race / count
- reward_rand / count
- reward_money
- reward_exp
- reward_cached.

Claim grants rewards, clears flags, sets inactive and increments hunting level by one.

### Persistence
All Hunting state/reward cache uses quest flags.
`PC::SetFlag` marks them for quest persistence; normal PC/character save flushes them.
Reward items/currency use their own item/player persistence paths, so reward claim is not a single atomic transaction.

### First verified bugs
- BUG-HUNT-001: action 2 mission type is unvalidated; arbitrary client value becomes fixed-table index.
- BUG-HUNT-002: action 4 has no reward/completion authorization and always advances mission level.
- BUG-HUNT-003: gold-cap rejection still clears cached gold reward.
- BUG-HUNT-004: final mission 90 claim advances level to 91 with no terminal guard; subsequent normal Hunting window paths can index beyond the defined 90-mission domain.

### Status
Hunting System: **PARTIAL**.

### Next
1. locate/validate all Hunting static reward/mission tables and their exact dimensions.
2. finish client packet parser + packet-info/sequence audit.
3. audit reward item grant failure/ground fallback and crash atomicity.
4. inspect quest-flag save/relogin lifecycle at reward boundaries.
5. check normal final-level 90 behavior end-to-end.

## Related
- Bugs: `../bugs/hunting.md`
- Runtime tests: `../tests/hunting.md`
- Full legacy archive: `../archive/00_PROGRESS.md`
