# Classic Pet System — Deferred Runtime Tests

**Execution:** LOCKED / NOT RUN

## PET-T01 — Bruce pickup Y-axis range

**Owner bug:** `BUG-PET-001`  
**Execution state:** NOT RUN / LOCKED

When runtime phase is explicitly opened:
1. equip/summon current Bruce item 53233;
2. place an owned drop with X delta inside 900 but Y delta greater than 900, while still inside the pet's sectree scan area;
3. compare with an equivalent X/Y radial placement;
4. observe whether Bruce selects/travels to the Y-distant item despite nominal 900 range;
5. verify normal owner/loot restrictions remain intact.

Expected static result: the Y-distant item can pass the range predicate because its Y delta is hard-coded to zero.

No Classic Pet runtime test has been executed.


## PET-T02 — Bruce stale pickup target lifetime

**Owner bug:** `BUG-PET-002`  
**Execution state:** NOT RUN / LOCKED

When runtime phase is explicitly opened:
1. equip/summon Bruce item 53233;
2. drop an owned gold item or stackable item far enough that Bruce starts moving toward it instead of instantly collecting;
3. after Bruce selects the target, manually pick it up before Bruce reaches it;
4. for the stack case, ensure it fully merges so the original ground object is destroyed;
5. observe the next pet update for crash/stale-target behavior;
6. repeat with a non-destructive inventory move as a control.

Expected static result: the destructive pickup path frees the object still cached by Bruce.

No test is authorized before the global runtime phase opens.


## PET-T03 — REAL_TIME PET_PAY expires while summoned

**Owner bug:** `BUG-PET-003`  
**Execution state:** NOT RUN / LOCKED

When runtime phase is explicitly opened:
1. use a controlled short-duration `ITEM_PET / PET_PAY` with the same `REAL_TIME` lifecycle as current deployed pets;
2. equip/summon it and keep the owner online through expiry;
3. confirm the summon item is removed by `REAL_TIME_EXPIRE`;
4. immediately check whether the pet actor remains visible/summoned and whether its pet-system update event continues;
5. verify cleanup after owner logout/destruction as the control boundary;
6. capture server logs/state for the missing summon-item update branch.

Expected static result: item expiry bypasses `PetUnsummon`; the actor remains summoned until a later teardown/cleanup path.

No test is authorized before the global runtime phase opens.
