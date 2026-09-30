# Blue Dragon / Beran Setaou — Deferred Runtime Tests

**Status:** STATIC MAPPING OPEN / 2 TESTS DOCUMENTED / EXECUTION LOCKED / NOT RUN

Runtime/fault-injection execution remains globally locked.

## BDL-T01 — disconnect between item removal and delayed entry timer
Start a new Blue Dragon entry, confirm the access items are removed, then disconnect before `dragon_lair_warptimer` fires.

Expected signature in the current code: the personal quest timer is canceled with the quest PC teardown, no entry/refund callback executes, and the removed access items remain lost.

Covers `BUG-BDL-001`.



## BDL-T02 — disconnect while entry NPC is locked
Begin the first-entry dialogue far enough for `npc.lock()` to succeed, then disconnect before an explicit `npc.unlock()` branch executes. Reconnect with a different character and attempt to start the same entry dialogue.

Expected signature in the current code: the NPC still holds the departed PID and subsequent `npc.lock()` calls return false for other players. A corrected lifecycle must release the lock on quest cancellation/logout.

Covers `BUG-BDL-002`.
