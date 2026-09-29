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
