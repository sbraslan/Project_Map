# Fishing Renewal — Static Map

**Status:** STATIC MAPPING IN PROGRESS  
**Phase:** Detection / Mapping Only  
**Source policy:** read-only source/game repositories; only Project_Map is writable.  
**Feature gate:** `ENABLE_FISHING_RENEWAL`.

## Entry roots
Server:
- `game/src/char_fishing.cpp` — renewed fishing start/stop/catch/event/reward lifecycle.
- `game/src/fishing.cpp/.h` — fish tables and `GetFishCatchedVnum()`.
- `game/src/input_main.cpp::FishingNew` — `HEADER_CG_FISHING_NEW` dispatch.
- `game/src/packet.h::TPacketFishingNew`.
- `game/src/char_item.cpp::USE_BAIT` — bait -> rod socket2.
- `game/src/item_manager.cpp::CreateItem/DestroyItem` — temporary-item lifetime boundary.

Client:
- `Project_Binary/root/uifishing.py` — renewed fishing minigame UI and catch/fail actions.

Data:
- `Project_DumpProto/tr/item_names.txt` — current rod family names/VNUM presence.

## Normal renewed flow
Client starts renewed fishing
-> `HEADER_CG_FISHING_NEW / FISHING_SUBHEADER_NEW_START`
-> `CInputMain::FishingNew`
-> `CHARACTER::fishing_new_start()`
-> validates map/rod/bait/inventory
-> chooses fish through `fishing::GetFishCatchedVnum(..., second)`
-> starts `m_pkFishingNewEvent`
-> client fishing UI sends catch/fail packets
-> three valid catches call `fishing_catch_decision(itemVnum)`
-> chance roll -> AutoGiveItem on success.

Current rod-name deployment includes:
- 27400..27490 = Olta +1..+10;
- 27500..27590 = Olta +11..+20;
- 27591 = Karbon olta.

The code defines `second = false` only for VNUM 27400..27490, so 27500..27591 use the second fish tables.

## BUG-FISH-001 — second normal fish table can be indexed out of bounds
`aFishSecondTableNormal` contains 5 entries, but the normal second-table path returns:
`aFishSecondTableNormal[number(0, 6)]`.

For second-family rods, indices 5 or 6 are therefore outside the declared array.

Reachability is normal gameplay:
`FishingNew START -> fishing_new_start -> GetFishCatchedVnum(..., second=true)`.

Consequence: undefined read / incorrect selected fish VNUM is possible during ordinary renewed fishing with current +11..+20 / Carbon rod family. A crash is not claimed statically.

Promoted as `BUG-FISH-001`.

## BUG-FISH-002 — fishing start leaks a temporary item object
`fishing_new_start()` calls:
`ITEM_MANAGER::CreateItem(50187)`
only to pass the created object to `GetEmptyInventory(...)`.

The temporary item is never added to the character and no `RemoveItem/DestroyItem/M2_DESTROY_ITEM` call releases it on either the success path or the inventory-full return path.

`ITEM_MANAGER::CreateItem` allocates a `CItem`, assigns ID/VID, and registers it in the manager maps when `bSkipSave == false`; explicit `DestroyItem` removes those registrations and deletes the object.

Current item names contain VNUM 50187 (Çırak Sandığı I), so the probe item exists in tracked deployment.

Consequence: each renewed fishing start can leave an ownerless registered item allocated, causing cumulative item-manager/memory growth.

Promoted as `BUG-FISH-002`.

## Additional candidate
In `fishing_catch_decision`:
`if (dwVnum == 27591 && dwVnum >= 27400 && dwVnum <= 27490)`
is impossible because 27591 cannot also be <=27490. The exact intended Carbon-rod chance rule still needs semantic/data closure before promotion.

## Exact next work
1. map packet registration/size checks on client and server;
2. audit catch/fail packet trust, timing and replay boundaries;
3. close bait/POINT_FISHING_RARE arithmetic and uint8 probability underflow/overflow;
4. trace success reward, logging and Battle Pass/Achievement ownership;
5. audit stop/death/warp/logout/equipment-change cleanup;
6. audit rod refine lifecycle and current proto values;
7. promote only verified reachable findings.

No Fishing runtime test is authorized. Global first future live gate remains `DUNGEON-T09`.
