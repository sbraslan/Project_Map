# Zodiac Temple / 12ZI — Deferred Runtime Tests

**Status:** STATIC MAPPING CLOSED / 9 TESTS DOCUMENTED / EXECUTION LOCKED / NOT RUN

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

## ZOD-T08 — alternate DecMember iterator invalidation
Set `zodiac_disconnect_member_2=1`, place a PC in a Zodiac instance so it exists in `m_set_pkCharacter`, then trigger the normal member-removal/disconnect path under an iterator-debug build and/or ASan/UBSan.

Expected signature: erasing the current set element is followed by incrementing the invalidated iterator; debug STL/sanitizer instrumentation should flag the invalid iterator operation or expose a crash/corrupted traversal.

Covers `BUG-ZOD-008`.

## ZOD-T09 — bead catch-up remainder preservation
Prepare a character with fewer than 36 beads and `12zi_temple.beadtime` approximately 7199 seconds in the past, then perform the login path that calls `BeadTime()`.

Expected signature in the current code: one bead is granted, `beadtime` is reset to the current second instead of preserving the ~3599-second remainder, and the immediate `Bead_time` command carries a negative value. A corrected implementation should preserve the modulo-hour progress and publish a non-negative interval to the next bead.

Covers `BUG-ZOD-009`.

Do not run any ZOD test until the user explicitly changes the execution phase.
