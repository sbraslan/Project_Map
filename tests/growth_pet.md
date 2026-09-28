# Growth Pet System — Deferred Runtime Tests

**Execution:** LOCKED / NOT RUN

## GPET-T01 — 55713 hatch-info table boundary
Covers `BUG-GPET-001`.

Future isolated debug/ASan validation:
1. use disposable tracked egg VNUM 55413, whose Value0 is 55713;
2. instrument the hatching handler immediately around `bPetInfo = dwPetVnum - 55701`;
3. verify the derived index is 12 while `PET_HATCH_INFO_RANGE` has 12 rows / valid indices 0..11;
4. do not continue a destructive production hatch after the boundary is observed;
5. separately, with a disposable 55713 upbringing seal, instrument the enabled PET_ATTR_DETERMINE path before its identical table lookup.

Static prediction:
both current-data paths attempt an out-of-bounds read at index 12.

Safety class: **Stage C memory-safety / ASan / disposable data only**.

Global first future live gate remains `DUNGEON-T10`.


## GPET-T02 — Oversized feed item count
Covers `BUG-GPET-002`.

Future isolated debug/ASan test:
1. send a Growth Pet feed packet with the normal fixed 9-slot array;
2. set `iFeedItemsCount` to 10 or another value above 9;
3. break/instrument before inventory mutation;
4. confirm `ItemCubeFeed()` indexes beyond `iFeedItemsCubeSlot[8]`.

Static prediction: server performs an out-of-bounds read.

Safety class: **Stage C crafted packet / ASan only**.

## GPET-T03 — Out-of-range skill slot index
Covers `BUG-GPET-003`.

Future isolated debug/ASan tests should exercise one request at a time with slot index 3 first:
- learn-skill;
- skill-upgrade preview;
- final skill upgrade;
- delete-skill.

Static prediction: each path indexes a fixed three-element skill array without a slot-range guard; final upgrade/delete can write outside the array.

Safety class: **Stage C crafted packet / ASan only**.

## GPET-T04 — Non-skill item as skill book / invalid Value0
Covers `BUG-GPET-004`.

Future isolated debug/ASan test:
1. use a disposable active pet that is allowed to learn a skill;
2. select a non-PET current item whose Value0 exceeds 23, e.g. tracked VNUM 25101 with Value0 100;
3. send it as the skill-book inventory cell to a valid skill slot;
4. instrument assignment of `skill_vnum` and the following `GiveBuff()` table access.

Static prediction: the item type is not rejected, skill VNUM 100 is stored, and `pet_skill_table[100][0]` is addressed out of bounds.

Safety class: **Stage C crafted packet / ASan / disposable data only**.

## GPET-T05 — Premium revive count/type trust
Covers `BUG-GPET-005`.

Future isolated destructive test:
1. use a dead disposable pet whose revive requirement is greater than one;
2. select a one-count disposable non-PET stack as the material;
3. send a premium-revive packet claiming the required count;
4. record target count before the subtraction and after `SetCount()`.

Static prediction:
- non-PET material passes the current predicate;
- claimed count passes;
- unsigned subtraction underflows;
- `SetCount()` clamps the huge value to `g_bItemCountLimit`;
- revival continues.

Safety class: **Stage C destructive crafted packet / disposable data only**.

## GPET-T06 — Evolution duplicate-slot alias
Covers `BUG-GPET-006`.

Future isolated modified-client test:
1. prepare a pet at a valid evolution boundary;
2. prepare exactly one required material stack with exactly its configured requirement;
3. repeat that same inventory cell in all expected evolution material positions;
4. submit the feed/evolution request;
5. record validation count, consumed items and evolution result.

Static prediction: the same required VNUM satisfies all logical requirement entries before consumption; first removal destroys the one stack; later duplicate cells are empty; `EvolvePet()` still executes.

Safety class: **Stage B modified-client / disposable materials**.

## GPET-T07 — Premium revive structured-state persistence
Covers `BUG-GPET-007`.

Future normal disposable-pet test:
1. snapshot socket0, `pet_birthday`, and `pet_max_time`;
2. perform a legitimate premium revive;
3. inspect in-memory item `TGrowthPetInfo`;
4. relog and inspect DB-backed values again.

Static prediction: socket0 is renewed, but the birthday/duration changes made to the local `info` copy are absent from the item and DB state.

Safety class: **Stage A state/persistence observation**.

## GPET-T08 — Specialist table reachability
Covers `BUG-GPET-008`.

Future deterministic debug/normal-flow validation:
1. choose one specialist entry that is first in a four-record aggregate and one that is not;
2. invoke `GetGrowthPetSkillSpecialistValue()` for both;
3. compare returned values with the declared specialist ranges.

Suggested contrast:
- Monkey JIJOONG_WARRIOR: group-first, should be discoverable;
- Monkey HEAL: declared but not group-first, predicted to return 0.

Safety class: **Stage A observation/instrumentation**.

## GPET-T09 — HEAL delta semantics
Covers `BUG-GPET-009`.

Future controlled normal-flow test:
1. record owner current/max HP;
2. trigger a Growth Pet HEAL auto-skill;
3. record configured `skill_spec`, computed `restore_hp`, PointChange amount and final HP.

Static prediction: PointChange receives an absolute target HP as a delta, causing the actual heal to exceed the intended configured amount.

Safety class: **Stage A gameplay observation**.

## GPET-T10 — Non-NUL-terminated pet-name packet
Covers `BUG-GPET-010`.

Future isolated ASan tests:
1. construct hatching and name-change packets with all `PET_NAME_MAX_SIZE + 1` name bytes nonzero;
2. keep packet size otherwise valid;
3. instrument the initial name-length validation.

Static prediction: unbounded `strlen()` scans beyond the fixed name field before the later bounded `strnlen()` path is reached.

Safety class: **Stage C crafted packet / ASan only**.

## GPET-T11 — Final evolution age/socket mismatch
Covers `BUG-GPET-011`.

Future normal disposable-pet observation:
1. hatch a normal pet;
2. record `pet_birthday`, `pet_max_time`, socket0 and socket1;
3. compare `GetPetAgeDays()` with `GetPetBirthday()`;
4. at evolution 3 / level >=80, inspect `CanIncreaseEvolvePet()` before 30 actual days have elapsed.

Static prediction: socket1 contains duration seconds and the helper interprets it as a timestamp, producing an age far above 30 days and satisfying the final evolution age gate.

Safety class: **Stage A state observation**.
