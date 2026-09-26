# Battle Field System — Deferred Validation Notes

**Phase:** Detection / Mapping Only
**Execution status:** NOT RUN — documentation only

These are future validation ideas only. No runtime/fault-injection execution is allowed in the current phase.

## BFIELD-T01 — out-of-map delayed exit
Covers BUG-BFIELD-001:
- from a normal non-Battle-Field map where `CanWarp()` is true, issue `/exit_battle_field`;
- expected static behavior: Battle Field exit timer is created and character is warped to empire start.

## BFIELD-T02 — out-of-map immediate dead-exit command
Covers BUG-BFIELD-002:
- from outside Battle Field, including while alive, issue `/exit_battle_field_on_dead 1`;
- expected static behavior: direct `ExitCharacter` path, immediate empire-start warp, Battle Field cooldown set.

## BFIELD-T03 — repeat-kill cooldown renewal
Covers BUG-BFIELD-003:
- kill the same victim once;
- verify kills inside 60 seconds are rejected;
- wait until the stored expiry passes;
- kill once and immediately repeat;
- expected static defect signature: subsequent immediate repeats are accepted because the existing map timestamp was never overwritten.

Do not execute these tests unless the user explicitly changes the project phase.


## BFIELD-T04 — active-feature build check
Future compile validation for BUG-BFIELD-004:
- build GAME with the current `ENABLE_BATTLE_FIELD` + `ENABLE_RANKING_SYSTEM` defines;
- verify the two unqualified `LoadRanking(RK_CATEGORY_BF)` sites fail symbol lookup unless an external/unmapped build injection supplies a wrapper.

## BFIELD-T05 — sparse weekly winner rollover
Future DB-isolated validation for BUG-BFIELD-005:
- prefill `log.battle_week` positions 1..3 with old winners;
- run a rollover with only one current `week_score > 0` row;
- inspect `battle_week` and winner cache;
- expected static defect signature: old positions 2/3 remain eligible.

Do not execute during the current detection/mapping phase.


## BFIELD-T06 — event-date command on wrong channel
Future validation for BUG-BFIELD-006:
- execute `battle_set_event` as implementor on a non-99 channel;
- inspect that channel's local Battle Field event info and channel 99's event info;
- expected defect: command succeeds locally but channel 99 scheduler remains unchanged.

## BFIELD-T07 — online weekly winner flag refresh
Future validation for BUG-BFIELD-007:
- keep an old winner and a newly promoted winner online across rollover;
- reload winner cache;
- inspect Battle rank affect flags before reconnect;
- expected defect: old winner retains stale flag, new winner lacks its new flag until reconnect/SetWeakRankingPosition.

## BFIELD-T08 — schedule seconds arithmetic
Future deterministic validation for BUG-BFIELD-008:
- evaluate GetOpenTime/GetCloseTime at a known HH:MM:SS with SS != 0;
- compare against exact target-time subtraction;
- expected same-day difference: +2*SS seconds.

Do not execute during the current detection/mapping phase.
