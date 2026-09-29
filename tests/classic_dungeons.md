# Classic Quest Dungeons — Deferred Runtime Tests

**Status:** STATIC MAPPING CLOSED / 4 TESTS DOCUMENTED / EXECUTION LOCKED / NOT RUN

Runtime/fault-injection execution remains globally locked.

## CLD-T01 — concurrent-channel Spider Baroness boss isolation
Run one Spider Baroness session on channel A and one on channel B. Spawn boss 2092 in A, record its VID, then spawn the boss in B and record its VID. Kill an egg in A after B has overwritten the global event flag.

Expected signature in the current code: channel A reads B's globally propagated `king_vid` rather than A's boss VID. The multiplier update either finds no matching local entity or affects a different local entity if the numeric VID is valid there. A corrected implementation must keep boss identity channel/run-local.

Covers `BUG-CLD-001`.



## CLD-T02 — Snow leader reconnect before timeout
Start Snow Dungeon, advance until `dungeon_enter == 1`, disconnect/logout the party leader, then reconnect/rejoin before the 5-minute `REJOIN_LIMIT_TIME` expires and continue playing past the original deadline.

Expected signature in the current code: the reconnect succeeds, but the old `snow_dungeon_leader_out_timer` is still registered; at its original deadline it schedules `snow_dungeon_end_timer`, which ejects the party about 10 seconds later. A corrected flow must cancel the leader-out timer when the leader validly returns.

Covers `BUG-CLD-002`.


## CLD-T03 — Catacomb private-map creation failure after item consumption
Give a party member exactly one valid Catacomb rag/golden-lock entry item and force the private-map creation step behind `d.new_jump_party` to fail in a controlled test environment.

Expected signature in the current code: the item is removed before the create attempt, no dungeon is entered, and no item is restored. A corrected transaction should consume only after successful instance creation/registration or compensate on failure.

Covers `BUG-CLD-003`.


## CLD-T04 — Snow stage transition under competing server-timer context
Run two Snow private instances (A and B) on the same game process. Ensure a server timer for B is the most recent callback, then trigger one of A's affected non-timer transitions (for example `LEVEL2_KEY.use`).

Expected signature in the current code: A registers the floor transition using the stale server-timer arg from B instead of A's `d.get_map_index()`. When the timer fires it selects B (or another stale map), leaving A stuck and potentially advancing the wrong instance. Repeat for the level-5 cube, level-6 stone and level-8 key paths.

Covers `BUG-CLD-004`.


## Static closure
`CLD-T01..CLD-T04` remain documented and **NOT RUN**. Runtime/fault-injection execution is still globally locked.
