# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active state:** Static Mapping Resumed
**Active subsystem:** Costume / Appearance / ChangeLook
**Status:** STATIC MAPPING IN PROGRESS
**Machine state:** `STATE.json`
**System:** `systems/costume_appearance.md`
**Bugs:** `bugs/costume_appearance.md`
**Tests:** `tests/costume_appearance.md`
**Item automation:** `ITEM_CREATION_AUTOMATION.md`
**Last updated:** 2026-09-28

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only. Runtime execution remains locked.

## Saved side workflow
When the user says `item ekleyeceğiz`, use `ITEM_CREATION_AUTOMATION.md` as the canonical generation workflow.

## Mapping progress
Costume / Appearance / ChangeLook is now open.

Mapped:
- body costume: item VNUM -> item_proto VALUE3 -> MSM ShapeIndex -> GR2/DDS;
- hair costume VALUE3 visual path;
- server ChangeLook packet/transaction flow;
- client/server ChangeLook eligibility logic;
- transmutation VNUM item-state propagation boundary.

Verified:
- `BUG-LOOK-001` — right-slot check-in before left target can null-deref server `CheckOtherItem`.
- `BUG-LOOK-002` — mount targets 50051..50053 accept arbitrary non-costume right materials; same faulty predicate exists client and server.

Deferred tests:
- `LOOK-T01` isolated modified-client server-safety validation;
- `LOOK-T02` UI eligibility + isolated disposable commit validation.

## Exact next work
1. close DB persistence for transmutation VNUM;
2. map ChangeLook clear-scroll/reset;
3. map mount appearance use/expiry;
4. inspect close/disconnect/raw-item lifetime boundaries;
5. promote findings only with static evidence.

DUNGEON-T10 remains prepared but runtime-locked; it is not being executed during renewed static mapping.

GitHub state is canonical.
