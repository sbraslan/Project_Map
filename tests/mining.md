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


## MIN-T03 — pickaxe substitution during delayed mining
After runtime is explicitly unlocked:
1. start mining with pickaxe A;
2. before event completion, replace it with valid pickaxe B without moving;
3. record which pick's refine grade affects ore chance;
4. force/observe a practice success and record which pick gains mastery.

Bug signature: B controls the completion and receives mastery.

## MIN-T04 — direct warp during active mining
After runtime is explicitly unlocked in an isolated same-process map setup:
1. start mining beside a valid vein;
2. trigger a direct warp without a movement packet;
3. keep the original vein alive;
4. observe whether the delayed event resolves at the destination and drops ore there.

## MIN-T05 — MINING_LOCATION logging reachability
After runtime is explicitly unlocked in an isolated environment, attempt mining requests at distances >1000 and >2500 and inspect hack logs.

Expected static result: both are rejected by the first >1000 return and `MINING_LOCATION` is never emitted.

Do not run while execution lock is active.


## MIN-T06 — Mining Event shutdown during active mining
After runtime is explicitly unlocked, in an isolated event-map test:
1. start mining a valid event vein;
2. stop the scheduled Mining Event while the player mining timer has <=10 seconds remaining;
3. verify the vein is in dead state but still manager-resolvable;
4. observe whether the pending player mining event still rolls ore and applies pickaxe practice before the vein's dead-event destroys it.

Bug signature: ore/mastery settles after event shutdown from a dead vein.

Covers `BUG-MIN-006`.

Do not run while execution lock is active.
