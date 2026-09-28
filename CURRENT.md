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
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime execution remains locked.

## Saved item workflow
`item ekleyeceğiz` -> use `ITEM_CREATION_AUTOMATION.md`.

## Closed this turn
- hide-costume + ChangeLook body/weapon visibility path: no bug verified;
- initial type/subtype/anti-flag compatibility logic mapped;
- raw checked-in item lifetime mapped through real-time expiry and destruction.

## Verified bug set
- `BUG-LOOK-001` — right-slot-before-left null dereference.
- `BUG-LOOK-002` — quest mount targets 50051..50053 accept arbitrary non-costume material.
- `BUG-LOOK-003` — CanWarp omits W_CHANGELOOK.
- `BUG-LOOK-004` — sealed right material lacks server revalidation.
- `BUG-LOOK-005` — LEFT can be removed/replaced after RIGHT compatibility check; Accept does not revalidate.
- `BUG-LOOK-006` — real-time expiry can delete checked-in item while CTransmutation keeps a dangling raw pointer.

## Deferred tests
LOOK-T01..LOOK-T06. None executed.

## Mount expiry
Still deferred as incomplete intent:
- helper/event code exists;
- Accept does not initialize socket2/start expiry;
- intended product semantics not yet sufficiently proven for promotion.

## Exact next work
1. audit `CTransmutation::Open` against all renewed open-window states;
2. close remaining mount expiry intent/call-chain;
3. inspect invalid cross-type client rendering from BUG-LOOK-005;
4. map costume item-creation companion dependencies;
5. consider subsystem closure only after these edges are exhausted.

DUNGEON-T10 remains prepared but runtime-locked.

GitHub state is canonical.
