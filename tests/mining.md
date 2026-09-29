# Mining / Pickaxe — Deferred Runtime Tests

**Execution status:** LOCKED / NOT RUN  
**Global first future live gate:** DUNGEON-T09

## MIN-T01 — pickaxe mastery/refine boundary
After runtime is explicitly unlocked:
1. use a deployed pickaxe whose socket0 equals Value2;
2. hand it to NPC 20015 through the deployed mining quest;
3. confirm quest offers refine;
4. record `__refine_pick` / `RealRefinePick` return.

Expected bug signature: quest reaches refine UI but C++ returns rejection because equality is not refinable.

Then, in an isolated test, raise socket0 to Value2+1 and confirm C++ accepts the boundary while the quest equality branch no longer opens.

## MIN-T02 — death during delayed mining
After runtime is explicitly unlocked:
1. start normal mining;
2. die before the delayed mining event fires without moving;
3. observe whether ore roll/drop and pickaxe practice still execute.

Do not run while execution lock is active.
