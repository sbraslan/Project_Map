# Growth Pet System — Deferred Runtime Tests

**Execution:** LOCKED / NOT RUN

## GPET-T01 — 55713 hatch-info table boundary
Covers `BUG-GPET-001`.

Future isolated debug/ASan validation:
1. use disposable tracked egg VNUM 55413, whose Value0 is 55713;
2. instrument the hatching handler immediately around `bPetInfo = dwPetVnum - 55701`;
3. verify the derived index is 12 while `PET_HATCH_INFO_RANGE` has 12 rows / valid indices 0..11;
4. do not continue a destructive production hatch after the boundary is observed;
5. separately, with a disposable 55713 upbringing seal, instrument the enabled PET_ATTR_DETERMINE path before its identical table lookup.

Static prediction:
both current-data paths attempt an out-of-bounds read at index 12.

Safety class: **Stage C memory-safety / ASan / disposable data only**.

Global first future live gate remains `DUNGEON-T10`.
