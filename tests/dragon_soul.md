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

## Historical candidate DS-C01 — PROMOTED
The first-item Step-refine equipped-state asymmetry is now canonical `BUG-DS-008 / DS-T08`.

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


## DS-T08 — Step refine equipped-first validation
Covers `BUG-DS-008`.

Future isolated modified-client/debug test:
1. prepare a valid Step refine set with one equipped Dragon Soul and the remaining matching materials/items;
2. place the equipped DS position in the packet grid;
3. record pointer ordering of the resolved item set;
4. only evaluate the branch where the equipped DS is `set_items.begin()`;
5. observe whether Step refine proceeds to consume/unequip that item.

Static prediction: the first pointer-sorted item bypasses `IsEquipped()`; later consumption can remove it.

Safety class: **Stage B/C modified-client + destructive lifetime/state test**.
Never use production items.

No Dragon Soul test has been executed.


## DS-T09 — Relog with persisted active DS set
Covers `BUG-DS-009`.

Future controlled normal-flow test:
1. use a disposable character with a complete active same-grade DS set and confirmed `NEW_AFFECT_DS_SET`;
2. populate relevant late wear slots (27..32) with ordinary equipment carrying known attributes;
3. record expected character points;
4. logout normally so affects persist;
5. login normally;
6. compare immediate post-login points against the mathematically expected ordinary equipment + DS base + DS set result;
7. trigger/observe a later full point recomputation and compare again.

Static prediction:
- login cleanup executes while active deck is still -1;
- uint8 wrap selects wear 27..32;
- any non-zero erroneous set-value subtraction produces temporary session stat drift until a full recomputation.

Safety class: **Stage A/B controlled normal-flow state observation**.
No packet crafting is required.

## DS-T10 — Pull-out count-1 extractor lifetime
Covers `BUG-DS-010`.

Future isolated debug/ASan test:
1. prepare an equipped disposable Dragon Soul;
2. prepare exactly one `EXTRACT_DRAGON_SOUL` extractor;
3. invoke the ordinary extractor-on-DS pull-out path;
4. cover either success or failure and record extractor destruction at `SetCount(0)`;
5. observe the later log formatting dereference of `pExtractor->GetVnum()`.

Static prediction: post-destruction extractor dereference in both pull-out outcome branches.

Safety class: **Stage C crash/lifetime/sanitizer**.
Never run on production.

## DS-T11 — Daily gift event with zero event ID
Covers `BUG-DS-011`.

Future isolated quest/configuration test:
1. use a disposable below-level and/or unqualified character that has never joined this event, so quest flag `event_id` is 0;
2. in an isolated environment, set an active `ds_dg_st/ds_dg_et` time window while leaving `ds_dg_id = 0`;
3. configure a harmless disposable gift item;
4. invoke the alchemist daily-gift chat;
5. record whether the level/qualification messages are skipped and the once-per-day gift path is reached.

Control case:
- repeat with a non-zero new `ds_dg_id`; eligibility checks should execute.

Safety class: **Stage B isolated quest/event configuration**.
Do not change production event flags.

## Dragon Soul readiness summary

Canonical ownership:
- DS-T01 -> BUG-DS-001
- DS-T02 -> BUG-DS-002
- DS-T03 -> BUG-DS-003
- DS-T04 -> BUG-DS-004
- DS-T05 -> BUG-DS-005
- DS-T06 -> BUG-DS-006
- DS-T07 -> BUG-DS-007
- DS-T08 -> BUG-DS-008
- DS-T09 -> BUG-DS-009
- DS-T10 -> BUG-DS-010
- DS-T11 -> BUG-DS-011

No Dragon Soul test has been executed.
The global first future live gate remains `DUNGEON-T09`.
