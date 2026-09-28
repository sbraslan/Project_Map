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
