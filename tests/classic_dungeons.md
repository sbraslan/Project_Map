# Classic Quest Dungeons — Deferred Runtime Tests

**Status:** STATIC MAPPING OPEN / 1 TEST DOCUMENTED / EXECUTION LOCKED / NOT RUN

Runtime/fault-injection execution remains globally locked.

## CLD-T01 — concurrent-channel Spider Baroness boss isolation
Run one Spider Baroness session on channel A and one on channel B. Spawn boss 2092 in A, record its VID, then spawn the boss in B and record its VID. Kill an egg in A after B has overwritten the global event flag.

Expected signature in the current code: channel A reads B's globally propagated `king_vid` rather than A's boss VID. The multiplier update either finds no matching local entity or affects a different local entity if the numeric VID is valid there. A corrected implementation must keep boss identity channel/run-local.

Covers `BUG-CLD-001`.

