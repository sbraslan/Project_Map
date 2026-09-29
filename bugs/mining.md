# Mining / Pickaxe — Bug Registry

## BUG-MINE-001 — pickaxe refine quest and C++ eligibility are mutually exclusive
**Class:** unreachable gameplay progression / dead upgrade path  
**Reachability:** VERIFIED — current `mining.quest` + registered `__refine_pick` binding.

### Static proof
- Quest invokes `__refine_pick` only at `socket0 == value2`.
- `__refine_pick` calls `mining::RealRefinePick`.
- `RealRefinePick` requires `Pick_Refinable`.
- `Pick_Refinable` returns false for every `socket0 <= value2`; it becomes true only when `socket0 > value2`.
- Quest treats every `socket0 != value2`, including `socket0 > value2`, as not ready.

### Consequence
The normal NPC 20015 quest path cannot successfully refine a pickaxe.

### Deferred validation
`MINE-T01`.

## BUG-MINE-002 — mining event survives same-character warp and can reward at destination
**Class:** lifecycle/location validation defect  
**Reachability:** VERIFIED for same-process/same-character warp paths.

### Static proof
- `CanWarp()` does not block active mining.
- `WarpSet()`, `WarpEnd()`, `Show()` and `Stop()` do not cancel `m_pkMiningEvent`.
- Mining completion re-resolves the original vein VID but does not compare player map/distance again.
- `OreDrop()` uses the player's current map and coordinates.

### Consequence
A mining operation started at one vein can survive relocation and, on success, create its ore drop at the destination map.

### Deferred validation
`MINE-T02`.

## BUG-MINE-003 — mining event survives death and can resolve rewards while dead
**Class:** lifecycle/state validation defect  
**Reachability:** VERIFIED.

### Static proof
- `Dead()` has no mining-event cancellation.
- Mining completion has no `IsDead()` check.
- Completion requires only a live character object, equipped valid pickaxe and still-resolvable source vein.

### Consequence
The event may perform success/reward and pickaxe-practice logic after the player has died.

### Deferred validation
`MINE-T03`.

## Candidate — raw ore is consumed before OreRefine internal Yang check
`OreRefine()` deducts 100 raw ore before checking `GetGold() < iCost`.

Current `guild_building_melt.quest` pre-checks the matching fee before calling the binding, so a straightforward legitimate insufficient-Yang path is currently guarded. Do not promote unless the remaining quest-yield/state audit proves a reachable balance change between the precheck and C++ transaction.
