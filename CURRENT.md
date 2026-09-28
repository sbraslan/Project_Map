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
Closed this turn:
- `dwTransmutationVnum` DB persistence is complete end-to-end; no persistence bug verified.
- ChangeLook reversal/clear scroll persistently resets the appearance; no clear-scroll bug verified.
- Character destruction closes `CTransmutation`.
- Client and server live ChangeLook prices both use 50M item / 30M mount.

Verified bug set:
- `BUG-LOOK-001` — right-slot-before-left server null dereference.
- `BUG-LOOK-002` — quest mount targets 50051..50053 accept arbitrary non-costume material.
- `BUG-LOOK-003` — `CanWarp()` omits `W_CHANGELOOK`; same-character warp can retain active transmutation state/raw item references.
- `BUG-LOOK-004` — sealed right material is blocked by official client but not revalidated server-side.

Deferred:
- mount ChangeLook expiry helper/call-chain inconsistencies;
- broad `IsExpireTimeItem()` predicate;
- alternate item-mutation paths while raw `LPITEM` references are retained.

Deferred tests:
- LOOK-T01..LOOK-T04; none executed.

## Exact next work
1. close mount ChangeLook use/expiry call-chain;
2. map hide-costume + ChangeLook interaction;
3. audit weapon/body/job/gender eligibility edge cases;
4. audit alternate mutation paths against raw transmutation pointers;
5. preserve source/game immutability.

DUNGEON-T10 remains prepared but runtime-locked.

GitHub state is canonical.
