# Snake Lair / Queen Nethis — Deferred Runtime Tests

**Status:** STATIC MAPPING CLOSED / 5 TESTS DOCUMENTED / EXECUTION LOCKED / NOT RUN

Runtime/fault-injection execution remains globally locked.

## SNK-T01 — wrong-order pillar key consumption
Reach the six-pillar floor with a valid 70422 key, then use it on pillar 2 before pillar 1.

Expected signature in the current code: the key disappears, the pillar remains locked, the progression counter is unchanged, and no replacement/refund occurs.

Covers `BUG-SNK-001`.

## SNK-T02 — wrong statue item consumption
Reach the statue floor and give a statue an unrelated item or the wrong elemental Snake statue item. Also repeat against a statue already marked complete.

Expected signature in the current code: the handed item is removed before target/element/block validation and no progression is awarded. A corrected implementation must validate first and consume only on success.

Covers `BUG-SNK-002`.

## SNK-T03 — multi-Siren floor completion
Force step-4 substep 11 to spawn 2, 3 or 4 Ice Sirens, then kill exactly one.

Expected signature in the current code: the first kill advances the instance to floor 5 even though other spawned Sirens remain. A corrected counter must require all spawned Sirens to be killed.

Covers `BUG-SNK-003`.


## SNK-T04 — destroy party during an active Snake instance
Create/start a Snake private instance, then trigger the server party-destruction path while members are connected inside.

Expected signature in the current code: players are warped out and the map index disappears from the Snake registry, but the private sectree remains allocated and its Snake events/entities continue until the original one-hour limit event fires. A corrected lifecycle should tear down or explicitly transfer ownership of the instance when registration is removed.

Covers `BUG-SNK-004`.


## SNK-T05 — disconnect during delayed Queen skill
Stay alive in an active Snake instance until the map-wide skill pulse schedules `m_pkSnakeSkillEvent`, then disconnect/destroy the character within the two-second delay under ASan/UBSan or an equivalent lifetime-debug build.

Expected signature in the current code: CHARACTER teardown does not cancel the queued Snake skill; when it fires, the raw saved character pointer is dereferenced. A corrected teardown must cancel the event or use a safe identity lookup/lifetime guard.

Covers `BUG-SNK-005`.


## Static closure
`SNK-T01..SNK-T05` remain documented and **NOT RUN**. Runtime/fault-injection execution is still globally locked.
