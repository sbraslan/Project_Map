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


## Checkpoint — Bruce pickup lifetime + PET_PAY removal boundary

### BUG-PET-002 — Bruce keeps a raw ground-item pointer across update ticks
`PetPickUpItemStruct` stores the selected target directly through:
`pet->SetPickupItem(item)`.

The actor then keeps that raw `LPITEM` while moving toward the target:
`CheckPetPickup()`
-> `PickUpItems(900)`
-> `BringItem()`.

If the item is farther than 250 units from the pet, `BringItem()` starts movement and leaves the pointer stored for later 4 Hz update ticks.

The player can still manually pick up the same owned ground item during that interval.

Reachable destruction cases in `CHARACTER::PickupItem()` / `PickupItemByPet()` include:
- gold/ELK: ground item is removed and `M2_DESTROY_ITEM(item)` is called;
- stackable item that fully merges into an existing stack: `M2_DESTROY_ITEM(item)`.

On the next pet update, `BringItem()` reads the cached pointer and immediately dereferences it for `item->GetX()/GetY()` without re-resolving by VID or validating ownership/sectree/liveness.

This is a reachable stale-pointer / use-after-free path and is promoted as `BUG-PET-002`.

### Raw pickup lifetime classification
If manual pickup merely moves the item into inventory without destroying it, the cached object remains allocated but no longer represents a ground target. The destructive gold/full-stack cases are sufficient to establish the stronger lifetime defect.

### PET_PAY forced-removal boundary
Normal user toggle is safe:
`PET_PAY use`
-> `UnequipItem(item)`
-> `PetUnsummon(item)`.

Forced item deletion is different:
`ITEM_MANAGER::RemoveItem`
-> `CItem::RemoveFromCharacter`
-> equipped item `CItem::Unequip()`
-> item destruction.

That lower-level path does not call `CHARACTER::PetUnsummon()`.

`CPetActor::Update()` also has a missing-summon-item branch that returns `false` before `Unsummon()`, while the pet-system event ignores the Update return value and continues scheduling.

This is a strong stale-pet candidate. Promotion is deferred until a concrete current PET_PAY forced-removal/expiry producer is closed from deployed data.

## Exact next work
1. close current PET_PAY real-time/timer-based expiry reachability and decide the stale-pet candidate;
2. audit CPetSystem event/actor-map teardown and owner logout/destruction;
3. locate/exclude active `pet.summon()` quest producers;
4. map current PET_PAY item/race/client coverage;
5. audit Achievement TYPE_SUMMON_PET accounting under missing-item and abnormal unsummon paths;
6. promote only verified reachable additional Classic Pet bugs/tests.

No runtime execution is authorized. Global first future live gate remains `DUNGEON-T10`.


## Checkpoint — REAL_TIME PET_PAY expiry reachability closed

Current decoded deployment data in `Project_DumpProto/tr/item_proto.txt` closes the forced-removal producer:
- many current `ITEM_PET / PET_PAY` rows carry `REAL_TIME` limits;
- Bruce item `53233` is `PET_PAY`, `WEAR_PET`, `REAL_TIME 2592000`, VALUE0/race `34055`;
- multiple 530xx/532xx/533xx classic pet families use the same real-time lifetime model.

Server expiry chain:
`CItem::StartRealTimeExpireEvent`
-> `real_time_expire_event`
-> `ITEM_MANAGER::RemoveItem(item, "REAL_TIME_EXPIRE")`.

Classic PET_PAY is not exempt from this expiry handler. The forced item-removal path unequips/destroys the item without calling `CHARACTER::PetUnsummon()`.

On the next classic-pet update:
- `CPetActor::Update()` cannot resolve the summon-item VID;
- it returns `false` before `Unsummon()`;
- `CPetSystem::Update()` only folds that return into its local result;
- `petsystem_update_event` ignores the result and schedules itself again.

Therefore an equipped/summoned real-time classic pet can outlive its expired summon item. This is promoted as `BUG-PET-003`.

### Teardown boundary
`CHARACTER::Destroy()` explicitly destroys `m_petSystem`. `CPetSystem::Destroy()` deletes actors and cancels the update event; actor destruction calls `Unsummon()`.

So the stale classic pet is not proven permanent across owner teardown. The verified defect window is after item expiry and before owner/pet-system destruction or another explicit cleanup path.

### Achievement follow-on candidate
`CPetActor::Unsummon()` resets `pet.summon_time` only inside the branch where the summon item can still be resolved. If stale-pet cleanup happens after REAL_TIME removed the item, the achievement flag can survive that cleanup. A later normal summon overwrites the flag, so the exact externally visible accounting consequence still needs closure before separate promotion.

## Updated next work
1. locate/exclude active `pet.summon()` quest producers and resolve the legacy Lua signature mismatch;
2. finish current PET_PAY item/race/client coverage;
3. close Achievement TYPE_SUMMON_PET abnormal-cleanup consequence;
4. audit remaining death/warp/login restoration edges;
5. promote only verified reachable additional bugs/tests.

No Classic Pet runtime test is authorized. Global first future live gate remains `DUNGEON-T10`.


## Checkpoint — legacy Lua producer excluded + Achievement time-loss closed

### Legacy Lua `pet.summon()` reachability
The compiled server still exposes the legacy quest API in `questlua_pet.cpp`, where the call shape is semantically mismatched against the active `ENABLE_PET_SYSTEM` C++ signature.

Current tracked deployment audit:
- GitHub code search across `Project_Game` finds no `pet.summon`, `pet.unsummon`, `pet.is_summon`, `pet.count_summoned` or `pet.spawn_effect` usage;
- current `share/locale/europe/quest/quest_list` contains no pet quest source;
- tracked active quest package therefore does not establish a live producer for the mismatched Lua call.

Decision: keep the signature mismatch as a dormant/deferred compatibility hazard; do not promote it as a current reachable Classic Pet bug.

### Achievement summon-time consequence
Current `Project_Game/share/locale/europe/achievements.xml` contains Achievement 60:
- comment: `Summon pet for 30 days`;
- task type = 4 / `TYPE_SUMMON_PET`;
- max_value = 2592000 seconds;
- restriction type 13 enables time-based accumulation.

Normal classic pet lifecycle:
- summon calls `CAchievementSystem::OnSummon(... TYPE_SUMMON_PET, vnum, 0, false)`;
- `pet.summon_time` stores summon start time;
- normal `CPetActor::Unsummon()` resolves the summon item, calls `OnSummon(... elapsed_time ...)`, then resets the flag.

Abnormal REAL_TIME expiry path from BUG-PET-003:
- summon item is destroyed before `Unsummon()`;
- later `CPetActor::Unsummon()` sees `pet.summon_time > 0` but cannot resolve `m_dwSummonItemVID`;
- elapsed duration is therefore not submitted to `CAchievementSystem::OnSummon`;
- the flag is also not reset in that branch;
- `CAchievementSystem::OnLogout()` later sees the stale flag and unconditionally resets it to 0 without crediting time.

This makes the lost accounting externally visible: real accumulated classic-pet summon duration can be discarded from the 30-day achievement after forced item expiry. Promoted as `BUG-PET-004`.

## Updated next work
1. finish current PET_PAY item/race/client coverage;
2. audit remaining death/warp/login restoration edges;
3. inspect pet auto-pickup ownership-expiry/re-target behavior for any additional reachable lifetime bug;
4. close Classic Pet static mapping when no further reachable candidates remain.

No Classic Pet runtime test is authorized. Global first future live gate remains `DUNGEON-T10`.
