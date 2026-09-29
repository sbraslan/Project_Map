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
