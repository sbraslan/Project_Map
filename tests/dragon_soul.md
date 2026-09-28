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

## Candidate DS-T04 — Step refine equipped-first asymmetry
Not yet tied to a verified bug.

Only in isolated modified-client testing:
- build a valid-count step-refine grid containing one equipped DS;
- vary allocation/pointer order;
- check whether equipped item becomes the first `std::set` element and bypasses the later `IsEquipped()` loop.

Do not run until the static candidate is promoted or explicitly selected.
