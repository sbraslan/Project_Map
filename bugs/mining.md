# Mining / Pickaxe — Verified Bugs

## BUG-MIN-001 — deployed pickaxe refine quest and C++ disagree on mastery boundary

**Class:** gameplay progression / unreachable normal refine path  
**Reachability:** VERIFIED through deployed `mining.quest` and registered `__refine_pick` binding.

### Proof
- Quest refine branch requires `socket0 == value2`.
- `__refine_pick` calls `RealRefinePick()`.
- `RealRefinePick()` rejects when `!Pick_Refinable()`.
- `Pick_Refinable()` rejects every value `<= value2`, so equality is rejected.
- `PracticePick()` can increment equality to `value2 + 1`, which C++ accepts, but the quest equality guard no longer opens.

### Consequence
The ordinary NPC quest path cannot hand a pickaxe to C++ at a mastery value that both layers accept.

### Deferred validation
`MIN-T01`.

## BUG-MIN-002 — delayed mining result can execute while player is dead

**Class:** lifecycle/state validation  
**Reachability:** VERIFIED from normal mining plus death before event completion.

### Proof
- Mining creates a delayed `m_pkMiningEvent`.
- Movement cancels it, but `Dead()` does not.
- `mining_event` has no `IsDead()` check.
- Event success can still call `OreDrop()` and `PracticePick()`.

### Consequence
A mining attempt begun while alive can finish and produce ore/mastery after the player has died.

### Deferred validation
`MIN-T02`.


## BUG-MIN-003 — pickaxe identity is not bound to the delayed mining attempt

**Class:** state integrity / progression manipulation  
**Reachability:** VERIFIED.

### Static proof
- mining event stores player PID and load VID only;
- event completion re-reads `GetWear(WEAR_WEAPON)`;
- current pick controls ore percentage and receives `PracticePick()`;
- `CanHandleItem()` does not block item handling for active mining;
- no equip/unequip path cancels mining.

### Consequence
A different pickaxe can be substituted after mining starts and before completion, changing the success calculation and redirecting mastery progress to the replacement pickaxe.

### Deferred validation
`MIN-T03`.

## BUG-MIN-004 — active mining can survive direct same-process warp

**Class:** lifecycle / location revalidation  
**Reachability:** VERIFIED for direct same-process warp paths.

### Static proof
- `CanWarp()` ignores `m_pkMiningEvent`;
- `WarpSet()` uses `Stop()`, not `OnMove()`;
- `Stop()` does not call `mining_cancel()`;
- delayed event does not re-check player/load map or distance;
- ore drop is created at the player's current map/position.

### Consequence
An attempt started beside a vein can resolve after the character has been directly warped elsewhere, provided the same character/event survives and the original vein remains resolvable.

### Deferred validation
`MIN-T04`.

## BUG-MIN-005 — distance-abuse HackLog is dead code

**Class:** anti-cheat observability defect  
**Reachability:** VERIFIED by control flow.

### Static proof
The function returns for distance >1000 before later testing distance >2500 for `HackLog("MINING_LOCATION")`.

### Consequence
Out-of-range mining attempts are rejected, but the intended explicit MINING_LOCATION telemetry can never record the >2500 case.

### Deferred validation
`MIN-T05`.
