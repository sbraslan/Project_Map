# Blue Dragon / Beran Setaou — Deferred Runtime Tests

**Status:** STATIC MAPPING OPEN / 1 TEST DOCUMENTED / EXECUTION LOCKED / NOT RUN

Runtime/fault-injection execution remains globally locked.

## BDL-T01 — disconnect between item removal and delayed entry timer
Start a new Blue Dragon entry, confirm the access items are removed, then disconnect before `dragon_lair_warptimer` fires.

Expected signature in the current code: the personal quest timer is canceled with the quest PC teardown, no entry/refund callback executes, and the removed access items remain lost.

Covers `BUG-BDL-001`.

