# Mining / Pickaxe — Bug Registry

## BUG-MINE-001 — pickaxe refinement is unreachable through the current quest threshold

**Class:** progression / quest-C++ contract mismatch  
**Reachability:** VERIFIED — current `quest_list` includes `n_npc/mining.quest`.

### Static proof
1. Current mining quest handles pick VNUM 29101..<29110.
2. The refine branch requires `item.get_socket(0) == item.get_value(2)`.
3. It calls `__refine_pick(item.get_cell())`.
4. The registered Lua function forwards to `mining::RealRefinePick`.
5. `RealRefinePick` calls `Pick_Refinable`.
6. `Pick_Refinable` rejects when `curExp <= maxExp`, so equality is rejected.
7. When `curExp > maxExp`, the quest's non-equality branch runs instead and never calls refine.

### Consequence
The normal deployed NPC quest cannot successfully reach the pickaxe-refine RNG path for +0..+8 pickaxes.

### Deferred validation
`MINE-T01`.

## BUG-MINE-002 — delayed mining completion ignores death and warp relocation

**Class:** lifecycle/state revalidation defect  
**Reachability:** VERIFIED.

### Static proof
1. Mining starts only near a valid vein and schedules a delayed event.
2. Ordinary movement cancels through `OnMove()->mining_cancel()`.
3. `Dead()` does not cancel the event.
4. `CanWarp()` does not block active mining.
5. `WarpSet()->Stop()` does not invoke `OnMove` or cancel mining.
6. Delayed `mining_event` does not check dead state, current map equality, or distance to vein.
7. If the vein still resolves by VID and a pick remains equipped, it performs the normal ore chance and `OreDrop`.
8. `OreDrop` uses the player's current map/coordinates.

### Consequence
Pending mining can resolve after death, and a pending session can resolve after warp with ore dropped at the destination map.

### Deferred validation
`MINE-T02`.
