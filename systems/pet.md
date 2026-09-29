# Classic Pet System — Static Map

**Status:** STATIC MAPPING IN PROGRESS  
**Phase:** Detection / Mapping Only  
**Source policy:** read-only source repos; only Project_Map may be edited.  
**Distinct from:** Growth Pet System.

## Active feature state
Server:
- `ENABLE_PET_SYSTEM` is enabled in `common/CommonDefines.h`.
- `PET_AUTO_PICKUP` is enabled under it.
- legacy gate `__PET_SYSTEM__` is also defined in `common/service.h`.
- `PetSystem.cpp` and `questlua_pet.cpp` are both in the game Makefile/project.

## Initial entry roots

Server:
- `game/src/PetSystem.cpp/.h` — CPetSystem / CPetActor summon, unsummon, follow, auto-pickup, event lifetime;
- `game/src/char.cpp::PetSummon/PetUnsummon/CheckPet` — active PET_PAY helpers;
- `game/src/char_item.cpp::ITEM_PET/PET_PAY` — equip/use lifecycle;
- `game/src/item.cpp` — PET_PAY equip slot and equipability;
- `game/src/questlua_pet.cpp` — legacy Lua pet API;
- `game/src/input_login.cpp` — login/check-pet restoration boundary;
- Achievement integration through `TYPE_SUMMON_PET`.

Data/client surfaces to close:
- packed item_proto PET_PAY families;
- pet race/npclist/assets;
- client PET_PAY slot/UI;
- item expiry/unequip/death/warp/logout;
- auto-pickup ownership and item lifetime.

## Initial lifecycle map

### Normal PET_PAY item path
`ITEM_PET / PET_PAY use`
-> if not equipped: `EquipItem(item)` then `CHARACTER::PetSummon(item)`
-> `PetSummon` reads item VALUE0 as mob VNUM
-> `CPetSystem::Summon(mobVnum, petItem, false)`
-> actor spawn + summon-item VID binding + owner `ComputePoints()`
-> update event runs at 4 Hz.

Using the equipped item:
-> `UnequipItem(item)`
-> `CHARACTER::PetUnsummon(item)`
-> actor unsummon.

`WEAR_PET` is the equip slot for `ITEM_PET/PET_PAY`.

### CPetActor update
A summoned actor:
- follows owner when `EPetOption_Followable`;
- unsummons on owner death, pet death, invalid/lost summon item, or unequipped summon item;
- for race 34055 (Bruce), runs PET_AUTO_PICKUP.

### Legacy Lua API compatibility candidate
`questlua_pet.cpp` still calls the pre-ENABLE_PET_SYSTEM four-argument summon shape:
`Summon(mobVnum, pItem, petName, bFromFar)`.

The active C++ signature is:
`Summon(mobVnum, pItem, bool bSpawnFar, uint32_t options=...)`.

C++ implicit conversion makes this compile:
- `petName` pointer -> `bSpawnFar`;
- `bFromFar` -> options bitmask.

This is a real semantic mismatch, but it is not promoted until a tracked active quest producer calling `pet.summon()` is established. The normal PET_PAY item path uses the correct three-argument helper.

## BUG-PET-001 — Bruce auto-pickup range ignores Y distance

Packed current item proto:
- item 53233 = Bruce;
- `ITEM_PET`;
- subtype `PET_PAY` (12);
- VALUE0 = race 34055.

Current item description explicitly identifies Bruce as the auto-pickup pet.

`CPetActor::Update()` calls `CheckPetPickup()` only for race 34055.

The pickup candidate filter computes:
`DISTANCE_APPROX(item->GetX() - player->GetX(), player->GetY() - player->GetY())`.

The second term is always zero. The intended range check therefore ignores Y displacement and evaluates only X displacement.

A ground item owned by the player can pass the nominal 900 range filter despite being far outside 900 units on the Y axis, provided it is still present in the surrounding sectree iteration.

Promoted as `BUG-PET-001`.

## Exact next work
1. trace Bruce pickup raw-item lifetime through manual pickup, stacking, deletion, warp and ownership expiry;
2. close PET_PAY equip/unequip/real-time expiry/death/login restoration;
3. audit CPetSystem update-event and actor map cleanup for stale pointers;
4. locate/exclude active `pet.summon()` quest producers and resolve the legacy Lua signature mismatch;
5. map current PET_PAY item families/race assets/client UI;
6. audit Achievement TYPE_SUMMON_PET time accounting under abnormal unsummon paths;
7. create additional bugs/tests only for verified reachable paths.

No Classic Pet runtime test is authorized. Global first future live gate remains `DUNGEON-T10`.
