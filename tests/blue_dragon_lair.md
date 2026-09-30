# Blue Dragon / Beran Setaou — Deferred Runtime Tests

**Status:** STATIC MAPPING CLOSED / 4 TESTS DOCUMENTED / EXECUTION LOCKED / NOT RUN

Runtime/fault-injection execution remains globally locked.

## BDL-T01 — disconnect between item removal and delayed entry timer
Start a new Blue Dragon entry, confirm the access items are removed, then disconnect before `dragon_lair_warptimer` fires.

Expected signature in the current code: the personal quest timer is canceled with the quest PC teardown, no entry/refund callback executes, and the removed access items remain lost.

Covers `BUG-BDL-001`.



## BDL-T02 — disconnect while entry NPC is locked
Begin the first-entry dialogue far enough for `npc.lock()` to succeed, then disconnect before an explicit `npc.unlock()` branch executes. Reconnect with a different character and attempt to start the same entry dialogue.

Expected signature in the current code: the NPC still holds the departed PID and subsequent `npc.lock()` calls return false for other players. A corrected lifecycle must release the lock on quest cancellation/logout.

Covers `BUG-BDL-002`.


## BDL-T03 — join after early Beran kill
Start a run, kill Beran before the 10-minute group-entry window ends, then use a second eligible character with valid access items and the current run entry code.

Expected signature in the current code: the second character's items are consumed and the character is warped into map 208 even though `dragon_lair_alive == 0` and the room has already been purged.

Covers `BUG-BDL-003`.


## BDL-T04 — low-HP skill-damage and regeneration factors
Reduce Beran below 31% HP and compare the resolved `hp_damage` / `hp_regen` factors with the configured final rows.

Expected signature in the current code: both lookups return 0 because the final rows are defined as min 30 / max 0. A corrected table should make the 0-30 interval reachable and return 20 for damage and 12 for regen.

Covers `BUG-BDL-004`.


## Static closure
`BDL-T01..BDL-T04` remain documented and **NOT RUN**. Runtime/fault-injection execution is still globally locked.


## BDL-T03 — verify lair boss identity from deployed regen
Start a fresh Blue Dragon run and inspect the entity created by `data/dungeon/dragon_lair.txt`.

Expected signature in the current tracked data: the regen attempts VNUM 2430, while the configured Beran-Setaou proto/quest/C++ logic expects 2493. A corrected deployment must spawn the same canonical VNUM consumed by quest and combat code.

Covers `BUG-BDL-003`.


## BDL-T04 — low-HP regeneration band
Spawn canonical Beran-Setaou VNUM 2493 with the tracked `BlueDragon.lua`, reduce it below 30% HP, and inspect one recovery tick with no regen-percent stone contribution.

Expected signature in the current code: `BlueDragon_GetRangeFactor("hp_regen", hpPct)` returns 0 for the fourth band instead of 12 because the configured range is reversed.

Covers `BUG-BDL-004`.


## BDL-T05 — paid entry after boss death inside group window
Start a run, kill Beran-Setaou before the group-entry window expires, then use a second eligible character with the correct shared entry code and required access items.

Expected signature in the current code: the second character's items are consumed and the character is warped into map 208 even though `dragon_lair_alive == 0` and the room has already been purged.

Covers `BUG-BDL-005`.


## Static closure
`BDL-T01..BDL-T05` remain documented and **NOT RUN**. Runtime/fault-injection execution stays globally locked.
