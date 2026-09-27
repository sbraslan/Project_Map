# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Static Mapping Closure
**Status:** STATIC MAPPING COMPLETE
**Machine state:** `STATE.json`
**Last updated:** 2026-09-27

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime/fault-injection execution does not start unless the user explicitly changes phase.

## Just closed
- Sung Mahi Tower is now **STATIC COMPLETE**.
- Final static sweep added item restrictions, three curse consumers, restart/re-entry behavior and dungeon regen scaling.
- Verified `BUG-SMT-006`: tower-only consumables 70390–70395 and 70405 have server handling, localized names and client icon mappings, but are absent from tracked EN/DE/TR item_proto text sources.
- Sung Mahi verified bugs are `BUG-SMT-001` through `BUG-SMT-006`.
- `INDEX.md` now shows every listed subsystem as STATIC COMPLETE.
- Production C++, Python, quest, map, config and game data remain untouched.

## Deferred Sung Mahi findings
These remain intentionally unpromoted until missing runtime producer/schema evidence exists:
- `pc.mailbox_reward` nullable mailbox pointer precondition;
- missing `smhgate_flower` server folder for 9100–9107 without a tracked runtime caller;
- `sung_mahi_ranking.player_login` versus character-name mailbox key semantics;
- malformed/out-of-range quest-produced floor values;
- `m_bDungeon_Difficulty` versus `dungeonLevel` synchronization.

## Runtime readiness prepared
- `tests/sung_mahi_tower.md` now defines SMT-T01..SMT-T06 for BUG-SMT-001..006.
- `RUNTIME.md` now includes the Sung Mahi validation cluster.
- Existing global first runtime gate remains Dungeon T10 once runtime execution is explicitly enabled.
- No runtime test has been executed.

## Exact next work
1. Continue consolidating completed-subsystem bug/test readiness from the existing runtime cursor without executing tests.
2. Keep the existing DUNGEON-T10 normal-path reproduction as the first live gate for a future runtime phase.
3. Keep every source/game repository immutable until the phase is explicitly changed.

GitHub state is canonical.
