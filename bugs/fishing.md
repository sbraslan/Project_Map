# Fishing Renewal — Static Bug Registry

**Status:** ACTIVE STATIC MAPPING

## BUG-FISH-001 — second normal fish table out-of-bounds selection

**Class:** bounds / reward-selection defect  
**Reachability:** VERIFIED — normal renewed fishing with second-family rods.

### Static proof
1. `aFishSecondTableNormal` has 5 entries.
2. `GetFishCatchedVnum(..., second=true)` normal branch indexes it with `number(0, 6)`.
3. Indices 5 and 6 exceed the declared array.
4. `fishing_new_start()` calls this function during normal start.
5. `second` is false only for rod VNUM 27400..27490.
6. Current item names contain rods 27500..27590 (+11..+20) and 27591 (Carbon), so those rods select `second=true`.

### Consequence
Undefined memory read can produce an unintended fish VNUM during ordinary renewed fishing. A deterministic crash is not established.

### Deferred validation
Canonical test: `FISH-T01`.

## BUG-FISH-002 — renewed fishing start leaks temporary item 50187

**Class:** item lifetime / memory-manager leak  
**Reachability:** VERIFIED — every accepted start reaches the probe allocation before rod/bait checks complete.

### Static proof
1. `fishing_new_start()` creates item 50187 only for an inventory-space probe.
2. The returned LPITEM is not added to the character.
3. No destroy/remove call releases it before success or early return.
4. `ITEM_MANAGER::CreateItem` allocates/registers the item by default.
5. `ITEM_MANAGER::DestroyItem` is the corresponding unregister/delete path.
6. Current item names contain 50187.

### Consequence
Repeated fishing starts can accumulate ownerless registered item objects and consume memory/item-manager entries.

### Deferred validation
Canonical test: `FISH-T02`.

## Open candidate
- Carbon-rod bonus condition combines `dwVnum == 27591` with `dwVnum <= 27490`, making that branch unreachable; intended gameplay effect still needs closure.


## BUG-FISH-003 — client-authoritative renewed fishing hit validation

**Class:** gameplay trust / minigame validation bypass  
**Reachability:** VERIFIED — normal renewed fishing event plus client-originated CATCH packets.

### Static proof
1. `uifishing.py` performs the actual fish-inside-target hit test.
2. On a visual hit, client sends `CatchFishingNew()`; miss sends `CatchFishingFailed()`.
3. Server packet dispatch accepts `FISHING_SUBHEADER_NEW_CATCH` and calls `fishing_new_catch()`.
4. That function checks only active event state and a one-second last-catch gate.
5. It increments the trusted catch counter directly.
6. At three catches, the server proceeds to `fishing_catch_decision()`.
7. No server-side geometric/minigame-state validation proves that each accepted CATCH corresponds to an actual UI hit.

### Consequence
A modified client can bypass the renewed fishing dexterity minigame and generate the required successful-hit count by sending accepted CATCH packets at the server's timing boundary.

This does not bypass the final server-side reward chance roll, but it removes the intended minigame-skill requirement.

### Deferred validation
Canonical test: `FISH-T03`.
