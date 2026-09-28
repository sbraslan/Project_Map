# Costume / Appearance / ChangeLook — Deferred Tests

**Phase:** Detection / Mapping Only  
**Execution:** LOCKED / NOT RUN

## LOOK-T01 — Right-slot-before-left server safety
Covers `BUG-LOOK-001`.

Future isolated/modified-client test:
1. open ChangeLook window normally;
2. send a valid ChangeLook ITEM_CHECK_IN targeting RIGHT slot before LEFT is populated;
3. observe server/core behavior.

Expected from static code: null dereference in `CheckOtherItem` before the null guard.

Safety class: **Stage B isolated / modified client**.
Do not run on production.

## LOOK-T02 — Quest mount accepts non-costume material
Covers `BUG-LOOK-002`.

Future ordinary/UI-first validation:
1. open mount ChangeLook mode;
2. place one of 50051..50053 in the left slot;
3. attempt a clearly non-costume inventory item in the right slot;
4. verify whether UI accepts it;
5. do not press Accept in a valuable/live inventory until isolated test data is used.

Static client/server predicate predicts right-slot acceptance.

Second isolated step, only after safe disposable test data:
- Accept and verify stored ChangeLook VNUM / consumed material.

Safety class:
- eligibility observation: ordinary/current UI;
- commit/consumption: isolated disposable data.

No test executed.
