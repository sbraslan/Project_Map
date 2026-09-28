# Dungeon Core — Deferred Validation Notes

**Phase:** Detection / Mapping Only
**Execution status:** NOT RUN — documentation only

These entries are future validation ideas. The current project phase does not permit runtime/fault-injection execution or source modification.

## DCORE-T01 — orphan private dungeon after rejected entry
Future validation for BUG-DUNGEON-001:
- call registered `d.join` from a solo context under the active party-required build;
- separately call `d.new_jump_guild` without a guild;
- verify a private dungeon/map is allocated but receives no member and no dead cleanup event.

## DCORE-T02 — SpawnMoveUnique multiplicative spawn
Future validation for BUG-DUNGEON-002:
- in an isolated valid dungeon area, invoke one `d.spawn_move_unique(key,vnum,from,to)`;
- count actual spawned entities;
- compare against `d.get_unique_vid(key)`;
- expected static defect signature: multiple entities may spawn while only one key mapping is retained.

## DCORE-T03 — multi-key stale unique alias
Future ASan/debug validation for BUG-DUNGEON-003:
- spawn one dungeon mob and obtain its VID;
- assign the same VID to two different keys through `d.set_unique`;
- invoke generic dungeon purge/direct destruction;
- access the remaining alias through a unique getter;
- expected defect signature: stale raw pointer/UAF.

A normal-death variant should use three aliases because `DeadCharacter` is reached once at death and once at later manager destruction.


## DCORE-T04 — duplicate unique key registry divergence
Future validation for BUG-DUNGEON-004:
- call `d.spawn_unique("same", ...)` twice in an isolated dungeon;
- verify two entities exist while `d.get_unique_vid("same")` identifies only the first;
- purge/kill the key and verify the second unique-marked mob remains.
- separately exercise `d.set_unique("same", secondVID)` after the key already points to another mob.

Do not execute during the current detection/mapping phase.


## Dungeon Core readiness consolidation — 2026-09-28
- No Dungeon Core runtime test was executed.
- DCORE-T01 -> BUG-DUNGEON-001.
- DCORE-T02 -> BUG-DUNGEON-002.
- DCORE-T03 -> BUG-DUNGEON-003.
- DCORE-T04 -> BUG-DUNGEON-004.
- DCORE-T01 is a registered-Lua failure-path lifecycle test.
- DCORE-T02/DCORE-T04 are controlled dungeon-script semantic/regression tests.
- DCORE-T03 is isolated ASan/debug raw-pointer lifetime validation.
- Primary legitimate/current-code candidates: DCORE-T01 and DCORE-T02.
- Dungeon Core remains distinct from Dungeon Info; the global first live gate is still Dungeon Info DUNGEON-T10.
- Unnumbered defensive candidates from the static audit remain unpromoted and receive no invented runtime bug mapping.
