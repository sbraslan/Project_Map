# Growth Pet System — Static Bug Registry

## BUG-GPET-001 — Current 55713 growth pet indexes past the 12-row hatch-info table

**Status:** VERIFIED STATIC / CURRENT-DATA REACHABLE

`PET_HATCH_INFO_RANGE` is declared:
`const float PET_HATCH_INFO_RANGE[12][6]`.

The hatching handler derives:
`const uint8_t bPetInfo = dwPetVnum - 55701;`
and then accesses:
- `PET_HATCH_INFO_RANGE[bPetInfo][1..2]` for lifetime;
- `PET_HATCH_INFO_RANGE[bPetInfo][3..4]` for skill count.

Current tracked proto contains:
- PET_EGG `55413` with Value0 `55713`;
- PET_UPBRINGING `55713`.

For 55713:
`55713 - 55701 = 12`,
but valid indices for a 12-row array are `0..11`.

No server range check occurs before those reads.

The enabled `ENABLE_PET_ATTR_DETERMINE` path has the same defect: after checking only ITEM_PET/PET_UPBRINGING, `CHARACTER::PetAttrChange()` indexes `PET_HATCH_INFO_RANGE[itemPet->GetVnum() - 55701]` for type/lifetime/skill-count generation. Upbringing VNUM 55713 therefore again indexes row 12 out of bounds.

The client hatching UI reads the egg's Value0 and sends the normal hatching request; it does not remove the 55413/55713 path.

**Impact:** the current 13th pet family can read beyond the hatch-info table during ordinary hatching and during attribute determination. Outcomes are undefined and can include invalid lifetime/skill parameters, memory-safety faults, or process instability.

**Runtime:** Stage C debug/ASan with disposable 55413/55713 data only. See `GPET-T01`.
