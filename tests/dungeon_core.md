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
