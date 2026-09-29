# Mining / Pickaxe — Deferred Runtime Tests

Runtime execution is locked. Do not run these until an explicit phase change. Global first live runtime gate remains `DUNGEON-T09`.

## MINE-T01 — pickaxe refine eligibility mismatch
On isolated data after runtime unlock:
1. prepare pickaxe 29101..29109 with `socket0 == Value2`;
2. give it to NPC 20015 through the current mining quest;
3. verify quest reaches `__refine_pick` but `RealRefinePick` rejects it;
4. separately reach `socket0 > Value2` through practice and verify the quest routes to its not-ready branch.

Covers `BUG-MINE-001`.

## MINE-T02 — warp during mining countdown
On isolated data:
1. start mining a valid ore vein;
2. keep the vein alive;
3. perform a same-process/same-character warp before the 10..30 second mining event resolves;
4. force/instrument a success if necessary;
5. inspect event survival and ore-drop map/coordinates.

Bug indicator: event survives and ore is created at the destination.

Covers `BUG-MINE-002`.

## MINE-T03 — death during mining countdown
On isolated data:
1. start mining with a valid equipped pickaxe;
2. die before completion while the source vein remains;
3. keep the character object in the normal dead state;
4. observe whether completion still rolls success, creates ore and runs pickaxe practice.

Covers `BUG-MINE-003`.
