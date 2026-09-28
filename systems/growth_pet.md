# Growth Pet System — Static Map

**Status:** STATIC MAPPING IN PROGRESS  
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

## Current verified findings
`BUG-GPET-001..BUG-GPET-011`.

## Remaining static work
1. finish summon/dismiss/death/real-time expiry/rewarp lifetime;
2. audit EXP table and item/mob EXP arithmetic to level 105;
3. audit active/passive skill execution beyond HEAL;
4. audit name-change normal unsummoned branch;
5. audit attribute determine/change state and material consumption beyond the 55713 OOB;
6. close DB pet-table save/load/delete ownership and orphan lifecycle;
7. close current pet race/proto/client asset coverage;
8. consolidate runtime readiness and decide STATIC COMPLETE.
