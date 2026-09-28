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


## LOOK-T03 — Warp with ChangeLook window active
Covers `BUG-LOOK-003`.

Future controlled test:
1. open ChangeLook normally;
2. check in at least the left target;
3. invoke a normal server-authorized same-core warp path;
4. observe whether warp succeeds;
5. after arrival inspect ChangeLook/UI state and attempt ordinary inventory movement;
6. capture whether CANCEL/reopen/reconnect is needed to restore item handling.

Static prediction:
- `CanWarp()` does not reject `W_CHANGELOOK`;
- `WarpSet()` does not clear `m_pkTransmutation`.

Safety class: **Stage A/B controlled normal-flow**, depending on available same-core warp path.

## LOOK-T04 — Sealed material server revalidation
Covers `BUG-LOOK-004`.

Future isolated modified-client test:
1. use a disposable sealed/bound item that is otherwise type/subtype/anti-flag compatible as the right material;
2. bypass the official client UI seal check and send normal ChangeLook check-in;
3. verify server right-slot acceptance;
4. do not commit against valuable data;
5. in an isolated disposable setup, optionally Accept and verify consumption.

Static prediction: server accepts because no `IsSealed()` validation exists in the transmutation path.

Safety class: **Stage B isolated / modified client**.

No test executed.
