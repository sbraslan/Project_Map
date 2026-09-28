# Growth Pet System — Static Map

**Status:** STATIC COMPLETE  
**Phase:** Detection / Mapping Only  
**Source policy:** read-only source repos; only Project_Map may be edited.

## Entry roots

Server:
- `game/src/GrowthPetSystem.cpp`
- `game/src/GrowthPetSystem.h`
- `game/src/PetSystem.cpp/.h`
- `game/src/questlua_pet.cpp`
- Growth Pet packet handling in `game/src/input_main.cpp`
- pet item-use integration in `game/src/char_item.cpp`

Client/UI:
- `Project_Binary/root/uipetinfo.py`
- pet hatch/feed/name-change UI scripts
- locale pet skill / premium feed data

Game/data:
- `Project_Game/share/locale/europe/pet_exp_table.txt`
- growth-pet race assets under `share/data/monster/*pet*`
- `Project_DumpProto/tr/item_proto.txt` PET_EGG / PET_UPBRINGING / pet-material definitions.

## Initial architecture

The growth-pet lifecycle is item-backed:
- PET_EGG item carries the target PET_UPBRINGING VNUM in Value0;
- hatching creates the PET_UPBRINGING seal item;
- the seal stores lifetime/pet identity in sockets and structured `TGrowthPetInfo`;
- summon creates a runtime pet actor owned by `CGrowthPetSystem`;
- leveling, evolution, feeding, skills, revive, attribute determine and name change mutate the item/DB-backed pet state.

Static limits observed:
- `PET_MAX_LEVEL = 105`;
- `PET_MAX_EVOLVE = 4`;
- `PET_MAX_SKILL_POINTS = 20`;
- `PET_MAX_FEED_SLOT = 9`.

## First verified data/code mismatch

`PET_HATCH_INFO_RANGE` is declared as exactly 12 rows, intended for upbringing VNUMs 55701..55712.

Both hatching and attribute-determination index it directly with:
`petVnum - 55701`
without a range check.

Tracked proto currently contains **13** PET_UPBRINGING items:
`55701..55713`.

Tracked PET_EGG data likewise contains egg `55413` whose Value0 is `55713`.

Therefore hatching egg 55413 computes table index 12 against a 12-row array (valid indices 0..11). The enabled PET_ATTR_DETERMINE path repeats the same out-of-bounds indexing for upbringing item 55713.

See `BUG-GPET-001`.

## Next static work
1. map hatch packet/UI -> server -> DB -> seal creation atomically;
2. map summon/dismiss/death/revive and lifetime persistence;
3. map mob/item EXP arithmetic and pet_exp_table boundaries;
4. map evolution materials/count/consumption;
5. map feed windows and item-slot trust boundaries;
6. map skill learn/upgrade/delete/passive/auto-skill execution;
7. audit PET_ATTR_DETERMINE and name-change inputs;
8. close disconnect/warp/owner destruction and DB lifetime;
9. map current 55701..55713 proto/race/client visual coverage;
10. create verified bugs/tests only for reachable current paths.

No runtime test is authorized. Global first future live gate remains `DUNGEON-T10`.


## Packet trust / skill / revive / evolution pass — 2026-09-28

### Feed request boundary
The client packet fixes the feed-slot array at 9 cells, but carries a separate client-controlled count. Server forwarding does not cap that count and `ItemCubeFeed()` iterates it directly.

See `BUG-GPET-002`.

### Skill state shape and packet trust
Persistent `TGrowthPetInfo` has exactly three skill entries across VNUM/level/spec/cooldown/formula arrays.

Learn/upgrade/delete packets carry byte-sized slot indexes. Server actor methods use those indexes without a common `<3` guard.

See `BUG-GPET-003`.

The learn path also never verifies the selected inventory object is a `PET_SKILL` item and never bounds `GetValue(0)` to `PET_SKILL_MAX`. Current unrelated proto entries include Value0 values far above 23, so an invalid persistent skill VNUM can reach `pet_skill_table[skill_vnum][0]`.

See `BUG-GPET-004`.

### Premium revive
Premium revive support is enabled and tracked proto contains PET_PREMIUM_FEEDSTUFF VNUM 55100.

Server material validation rejects only wrong ITEM_PET subtypes rather than requiring ITEM_PET/PET_PREMIUM_FEEDSTUFF.

Quantity validation uses the packet's claimed count, then subtracts the requirement from the actual item count. Because the count arithmetic is unsigned and `CItem::SetCount()` clamps huge values to `g_bItemCountLimit`, an undersized arbitrary stack can be inflated after underflow.

See `BUG-GPET-005`.

The helper `Revive()` separately mutates a local copy of `TGrowthPetInfo` for birthday/duration but never writes it back. Socket0 renewal survives; intended structured age metadata does not.

See `BUG-GPET-007`.

### Evolution material identity
Server evolution requirements are represented as a seven-entry VNUM->count map. Validation counts matching input positions rather than distinct fulfilled requirement keys. Duplicate cells/VNUMs are not rejected.

Repeating one exact-count required stack in all seven logical positions can therefore satisfy the validation count before consumption; after the first removal empties that cell, later duplicate positions disappear and evolution still executes.

See `BUG-GPET-006`.

### Specialist skill table
`pet_skill_specialist_table` is declared `[][4]` but populated as a flat list and looked up via `for (auto table : ...)` plus `table->field`. Only the first struct of each four-entry aggregate is examined.

See `BUG-GPET-008`.

### HEAL skill
HEAL computes an absolute target HP then sends that absolute value to delta-based `PointChange(POINT_HP, amount)`.

See `BUG-GPET-009`.

### Pet-name packet strings
Both hatching and name-change first run unbounded `strlen()` over fixed network character arrays before any bounded `strnlen()`.

See `BUG-GPET-010`.

### Birth socket / final evolution age
Hatching stores the pet's duration seconds in socket1. The final-evolution helper later treats socket1 as an absolute birth timestamp. This makes normally hatched pets appear far older than 30 days for the evolution-3 age condition.

See `BUG-GPET-011`.

## Closure pass — summon / expiry / EXP / warp / assets / transport box

### Summon, dismiss and expiry lifetime
`CGrowthPetSystemActor::Summon()` retains the active PET_UPBRINGING seal in `m_pkPetSeal`, copies its structured state, marks it summoned and locks the seal.

`Dismiss()` is the canonical release path:
- clears the Growth Pet buff;
- marks actor state unsummoned;
- unlocks the retained seal;
- writes actor `m_PetInfo` back to that seal;
- saves it;
- nulls `m_pkPetSeal`;
- destroys the spawned pet character.

Growth Pet/PET_BAG real-time expiry is special-cased in `real_time_expire_event`: the item is not removed when socket0 expires. The actor's periodic `Update()` then detects the dead seal and calls `Dismiss()`. Character destruction also destroys the GrowthPetSystem.

No additional expiration-driven retained-item UAF was promoted from this lifecycle.

### EXP boundary through level 105
The tracked `exp_pet_table_common` has entries 0..105. `GetNextExpFromTable()` indexes it only for levels <=105, and `SetExp()` rejects new EXP once `GetPetLevel() >= PET_MAX_LEVEL`.

Thus 104->105 is valid and the next EXP transaction is stopped before a level-106 table access.

The monster/item requirement relation is internally coherent:
- monster portion uses the table value;
- item portion is table/9;
- together this implements the expected 90/10 split.

No level-105 EXP OOB or arithmetic defect was promoted.

### Rewarp / owner lifecycle
`ENABLE_PET_SUMMON_AFTER_REWARP` is not enabled in the tracked build.

For a surviving same-core owner transition, follow AI relocates a pet that becomes distant by calling `Show(owner map, owner position...)`. Cross-core/logout destruction tears down GrowthPetSystem and dismisses active actor state.

No additional rewarp lifetime defect was promoted.

### Current item/race/client coverage
Tracked item-name data contains the current 13-family sequence:
- eggs `55401..55413`;
- upbringing seals `55701..55713`.

Client `npclist.txt` contains the mapped young/hero race pairs for the established families, including monkey, spider, Razador, Nemere, blue/red dragon, Azrael, executioner, Exedyar, Alastor/white-dragon, Baashido and Nessie families.

The 13th family is already proven current by `BUG-GPET-001`: VNUM 55713 reaches the hatch/determine table while the table has only 12 rows.

No separate asset-path defect was promoted beyond that current-data mismatch.

### Transport Box
PET_BAG handling adds two independent defects:
- `BUG-GPET-019`: successful bagging removes/destroys the target seal then reads `item2->GetName()`;
- `BUG-GPET-020`: bagging accepts an already dead Growth Pet and unbagging creates a new seal with `now + pet_max_time`, bypassing the revive path.

### Cross-system ownership
The generic player item-destroy path also dereferences a destroyed item name. That is already canonical `BUG-ITEM-001` and is not duplicated in Growth Pet ownership.

## Current verified findings
`BUG-GPET-001..BUG-GPET-020`.

## Runtime ownership
`GPET-T01..GPET-T20` are canonical deferred tests.

**Execution state:** READY / EXECUTION LOCKED / NOT RUN.

The global first future live gate remains `DUNGEON-T10`; Growth Pet runtime tests do not change that ordering.

## Static closure
Growth Pet System is **STATIC COMPLETE** for the tracked source/data snapshot.

Closure includes:
- hatch and name packet boundaries;
- feed/evolution packet shape and material identity;
- skill index/table/formula execution;
- attribute determine/change state;
- revive and lifetime persistence;
- summon/dismiss/death/expiry lifecycle;
- level 1..105 EXP arithmetic;
- Growth Pet DB row lifecycle;
- current 55701..55713 family/data coverage;
- PET_BAG lifecycle.

No source/game repository was modified.
