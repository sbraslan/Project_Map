# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Dragon Soul / Alchemy
**Status:** STATIC MAPPING IN PROGRESS
**Machine state:** `STATE.json`
**System:** `systems/dragon_soul.md`
**Bugs:** `bugs/dragon_soul.md`
**Tests:** `tests/dragon_soul.md`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime, crafted-packet, crash, sanitizer and fault-injection execution remains locked.

## Verified this opening pass
- `BUG-DS-001`: breaking an active complete DS set can leave stale set-bonus stats because cleanup returns on the first missing slot.
- `BUG-DS-002`: Dragon Heart extraction destroys count-1 source DS, then passes the dangling pointer to `ItemLog()`.
- `BUG-DS-003`: strength-refine success removes source from character before SetCount(0), leaving an ownerless zero-count CItem registered in memory while DB deletion is merely queued.

## Candidate / not promoted
- step refine first std::set item skips equipped-state validation;
- GetBasePosition grade == max boundary;
- change-attr step index lacks explicit bound.

## Next static work
1. material-count/stack semantics across all refine modes;
2. server/client dragon_soul_table dimension parity;
3. set-bonus reactivation/relog cleanup;
4. extraction aliasing and window/warp state;
5. qualification/daily quest lifecycle.

No runtime test executed.

Global first future live gate remains `DUNGEON-T10`; opening Dragon Soul static mapping does not authorize or reorder runtime execution.

GitHub state is canonical.
