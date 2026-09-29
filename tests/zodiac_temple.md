# Zodiac Temple / 12ZI — Deferred Runtime Tests

**Status:** STATIC MAPPING OPEN / 7 TESTS DOCUMENTED / EXECUTION LOCKED / NOT RUN

These are documentation-only test specifications. No runtime/fault-injection test is authorized while the global execution lock is active. Global first future live gate remains `DUNGEON-T09`.

## ZOD-T01 — temporary item object growth
Observe item-manager object counts/VID map while issuing valid-range `/cz_check_box` commands with insufficient materials, then exercise a valid `/cz_reward` path.

Expected signature: +2 ownerless registered objects per check-box attempt and +1 temporary object per reward call unless another cleanup path is discovered.

Covers `BUG-ZOD-001`.

## ZOD-T02 — asymmetric gold reward
Prepare yellow/green reward counters as 1/0 (and separately 0/1), ensure no 33028 stack exists in scanned inventory, then invoke `/cz_reward 3`.

Expected signature: paired count computes to zero, counters remain asymmetric, but one 33028 is created.

Covers `BUG-ZOD-002`.

## ZOD-T03 — duplicate check-box replay
Start a color mask at zero, satisfy one cell's row/column costs twice, and submit the same color/index twice through the server command.

Expected signature: the mask changes from 0 -> 1 -> 2 for index 0 instead of remaining 1.

Covers `BUG-ZOD-003`.

## ZOD-T04 — cross-instance revive boundary
Run two separate Zodiac private instances on the same game core, kill a target in instance B, and invoke the target VID from instance A.

Expected signature: `/revivedialog` resolves the unrelated target; `/revive` accepts both maps merely because they are in the Zodiac private-map range and revives B from A.

Covers `BUG-ZOD-004`.

## ZOD-T05 — first Zodiac skill event handle
Under instrumentation/sanitizer, create a fresh Zodiac combat character and trigger each delayed Zodiac skill before any corresponding event field has been assigned.

Expected signature: indeterminate `m_pkZodiacSkillN` read/cancel path is observable; sanitizer/compiler diagnostics may fire before event creation.

Covers `BUG-ZOD-005`.

## ZOD-T06 — pending-event teardown
Schedule a delayed Zodiac mob skill and destroy the mob/private map before callback. Separately schedule skill 11 and disconnect/destroy the victim before callback.

Expected signature: uncancelled callback retains a stale raw character pointer; sanitizer should identify the UAF if the stale address is dereferenced.

Covers `BUG-ZOD-006`.

## ZOD-T07 — manager initialize non-channel-99
With sanitizers/compiler diagnostics, execute Zodiac manager initialization on a channel other than 99.

Expected signature: control reaches the end of a non-void `bool` function with no return value.

Covers `BUG-ZOD-007`.

Do not run any ZOD test until the user explicitly changes the execution phase.
