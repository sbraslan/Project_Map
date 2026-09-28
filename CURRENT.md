# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
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

## This turn
- renewed W_* registry audited against `CTransmutation::Open()`;
- ChangeLook omits Acce/Aura/GuildBank/Roulette/Switchbot from its open guard;
- Acce overlap promoted to `BUG-LOOK-007`;
- cross-type rendering from `BUG-LOOK-005` statically closed:
  - PART_MAIN reads arbitrary ChangeLook item VALUE3 as ShapeIndex;
  - PART_WEAPON receives arbitrary final ChangeLook VNUM without type normalization;
- item creation automation updated with ChangeLook compatibility constraints.

## Verified bug set
`BUG-LOOK-001..007`.

## Deferred tests
`LOOK-T01..LOOK-T07`. None executed.

## No new bug
- hide-costume + ChangeLook remains sound;
- DB persistence/reversal remains sound;
- Aura overlap exists but no separate corruption root cause promoted beyond documented cross-window risk;
- GuildStorage/Roulette/Switchbot omissions need concrete ChangeLook mutation evidence before promotion.

## Exact next work
1. close mount expiry semantics;
2. inspect free-ticket pointer lifetime and duplicate-position aliasing;
3. finish omitted-window concrete-impact audit;
4. if no further verified edge remains, prepare STATIC COMPLETE + runtime-readiness documentation.

DUNGEON-T10 remains prepared but runtime-locked.

GitHub state is canonical.
