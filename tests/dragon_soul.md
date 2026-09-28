# Dragon Soul / Alchemy — Deferred Tests

**Phase:** Detection / Mapping Only  
**Execution:** LOCKED / NOT RUN

## DS-T01 — Break active complete set and compare set-bonus points
Covers `BUG-DS-001`.

Future controlled normal-path test:
1. equip a complete same-grade Dragon Soul set that activates `NEW_AFFECT_DS_SET`;
2. record only the stats contributed by the DS set;
3. pull out one stone through the normal UI;
4. confirm the set marker disappears;
5. compare character points against the expected no-set state.

Static prediction:
- cleanup returns on the first empty DS slot;
- part or all of the set bonus remains in character points.

Safety class: **Stage A normal-path state observation**.
Use disposable/non-production character data.

## DS-T02 — Dragon Heart extraction lifetime
Covers `BUG-DS-002`.

Future isolated debug/ASan test:
1. use a disposable count-1 Dragon Soul;
2. invoke Dragon Heart extraction;
3. cover both a successful charge result and, if controllable, a zero-charge result;
4. observe `SetCount(0)` destruction followed by `ItemLog` dereference.

Static prediction: use-after-free at the post-consumption ItemLog call.

Safety class: **Stage C crash/lifetime/sanitizer**.
Do not run on production.

## DS-T03 — Strength-success orphan item registry
Covers `BUG-DS-003`.

Future controlled debug test:
1. record source DS item ID/VID;
2. perform a successful strength refine with disposable materials;
3. verify source disappears from player inventory and DB destroy is queued;
4. inspect server item manager for the old ID/VID after delayed-save processing;
5. repeat to measure object-count growth.

Static prediction:
- source owner becomes nullptr before SetCount(0);
- DB delete occurs;
- in-memory source object remains registered.

Safety class: **Stage B controlled instrumentation / memory-registry observation**.

## Candidate DS-C01 — Step refine equipped-first asymmetry
Not yet tied to a verified bug.

Only in isolated modified-client testing:
- build a valid-count step-refine grid containing one equipped DS;
- vary allocation/pointer order;
- check whether equipped item becomes the first `std::set` element and bypasses the later `IsEquipped()` loop.

Do not run until the static candidate is promoted or explicitly selected.


## DS-T04 — Change Attribute material subtype enforcement
Covers `BUG-DS-004`.

Future isolated modified-client test:
1. open/authorize a Dragon Soul refine session;
2. use a disposable Myth DS eligible for attribute change;
3. submit MATERIAL_DS_REFINE_NORMAL/BLESSED/HOLLY as the material instead of MATERIAL_DS_CHANGE_ATTR;
4. verify server acceptance/consumption and resulting attribute reroll.

Static prediction: server accepts because all four material subtypes pass IsDragonSoulRefineMaterial.

Safety class: **Stage B modified-client / disposable data**.

## DS-T05 — Missing RefineStepTables validation
Covers `BUG-DS-005`.

Future isolated configuration test only:
1. copy the Dragon Soul table into a disposable test environment;
2. remove RefineStepTables while keeping RefineStrengthTables;
3. start under debugger/ASan;
4. observe CheckRefineStepTables reaching GetRefineStepValues with a null m_pRefineStepTableNode.

Static prediction: wrong guard does not reject the missing step node.

Safety class: **Stage C startup/configuration isolation**.
Never modify production table data for this test.

## DS-T06 — Normal-refine opener used for Change Attribute
Covers `BUG-DS-006`.

Future isolated modified-client test:
1. invoke the normal GM_PLAYER `/refine_open` path;
2. do not open the dedicated Change Attribute window;
3. send a valid `DS_SUB_HEADER_DO_CHANGE_ATTR` grid for disposable eligible data;
4. verify that the server processes it because opener != nullptr.

Also record qualification state to confirm the normal command does not add a separate qualification gate.

Safety class: **Stage B authorization / modified client**.

## DS-T07 — Warp while Dragon Soul refine opener is active
Covers `BUG-DS-007`.

Future controlled observation:
1. open the normal Dragon Soul refine window;
2. invoke an ordinary server-authorized warp without manually closing the window;
3. after arrival, attempt a normal inventory operation;
4. record whether CanHandleItem remains blocked until a refine CLOSE/reconnect clears the opener.

Static prediction: CanWarp/WarpSet preserve the opener pointer.

Safety class: **Stage A/B normal-flow state observation**.

No Dragon Soul test has been executed.
