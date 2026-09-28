# Aura System — Static Map

**Status:** STATIC COMPLETE  
**Phase:** Detection / Mapping Only  
**Source policy:** read-only source repos; only Project_Map may be edited.

## Entry points mapped so far

Server implementation:
- `Project_ServerSRC/game/src/char_aura.cpp`
- `CHARACTER::OpenAuraRefineWindow()`
- `CHARACTER::AuraRefineWindowCheckIn()`
- `CHARACTER::AuraRefineWindowCheckOut()`
- `CHARACTER::AuraRefineWindowAccept()`
- `CHARACTER::IsAuraRefineWindowCanRefine()`

State:
- `m_pointsInstant.m_pAuraRefineWindowOpener`
- `m_bAuraRefineWindowType`
- `m_bAuraRefineWindowOpen`
- `m_pAuraRefineWindowItemSlot[]`
- `W_AURA` under ENABLE_CHECK_WINDOW_RENEWAL

Window types currently observed:
- ABSORB
- GROWTH
- EVOLVE

The server stores Aura checked-in items as `TItemPos` and locks the real inventory item while it is present in the window.

## First verified static finding

### Aura distance authorization is bypassed after opening
`IsAuraRefineWindowCanRefine()` intends to enforce:
1. global item-handling eligibility;
2. Aura window open;
3. non-null opener;
4. distance < `AURA_REFINE_MAX_DISTANCE`.

But it begins with:
```
if (!CanHandleItem())
    return false;
```

The generic `CanHandleItem()` itself returns false while Aura is open or has an opener.

Therefore, during the exact state in which Aura operations are supposed to execute, `IsAuraRefineWindowCanRefine()` returns false before reaching the distance check.

Check-in, check-out and final accept all handle that false result like this:
- if Aura is open and opener is non-null, continue anyway;
- otherwise return.

That fallback converts the failed permission check into an allow path and bypasses the intended distance check.

See `BUG-AURA-001`.

## Cross-system observation
Aura and Dragon Soul do not share one universal opener mutex. Dragon Soul closure found no duplicate DS-specific destructive alias because Aura slots accept Aura-costume / armor / Aura-resource inputs rather than ITEM_DS. Aura-side overlap still requires its own audit.

## Exact next work
1. map client -> packet -> server Aura open/check-in/check-out/accept contract;
2. trace ABSORB item copy and material destruction lifetime;
3. trace GROWTH refine table, material counts, EXP/socket arithmetic and output persistence;
4. trace EVOLVE success/failure item lifecycle;
5. audit booster/eraser and absorption-rate arithmetic;
6. audit warp/disconnect/close cleanup and locked-item recovery;
7. audit opener lifetime and cross-window coexistence;
8. map Aura visual/proto/client persistence surfaces;
9. create additional bug/test records only from verified reachable paths.

No Aura runtime test is authorized. Global first future live gate remains `DUNGEON-T10`.


## Transaction contract / material validation pass — 2026-09-28

### Client -> packet -> server
The normal client keeps Aura UI policy checks in `uiaura.py`, then sends:
- check-in with inventory `TItemPos`, Aura slot and Aura window type;
- check-out with Aura slot and Aura window type;
- accept with Aura window type;
- cancel/close.

Unlike Acce, Aura check-in is authoritative server state:
- the real item is resolved;
- it is locked;
- its `TItemPos` is stored in `m_pAuraRefineWindowItemSlot[]`;
- Accept later resolves those stored positions again.

The server also binds the request to `m_bAuraRefineWindowType`, so a packet cannot simply change ABSORB/GROWTH/EVOLVE mode after opening.

The post-open distance authorization remains defective as `BUG-AURA-001`.

### ABSORB lifecycle
Main slot:
- requires `ITEM_COSTUME / COSTUME_AURA`;
- requires empty absorbed-item socket.

SUB slot:
- requires armor subtype shield/wrist/neck/ear.

Accept:
- stores material original VNUM in Aura socket 0;
- copies normal attributes and Yohara random applies;
- updates the Aura item;
- removes the absorbed material;
- clears the two server Aura slots.

No post-removal material dereference was found in the mapped ABSORB accept sequence.

The normal client also rejects wedding items in the SUB slot. The server currently has no equivalent wedding-item policy check; this remains a policy candidate until current tracked wedding-item classes/reachability are established.

The ABSORB result-preview packet has a coding typo under `ENABLE_SET_ITEM`: it writes `sub.pItem.set_value` instead of `sub2.pItem.set_value`. Current client Aura packet handling does not copy/use `set_value` from this packet, so no user-visible/runtime consequence is promoted from that typo.

### GROWTH table / EXP arithmetic
Aura refinement bands are:
- level 1..49: need EXP 1000;
- 50..99: 2000;
- 100..149: 4000;
- 150..199: 8000;
- 200..249: 16000;
- 250 radiant terminal row.

GROWTH does not cross a grade boundary: at the current row's `LEVEL_MAX`, reaching full required EXP stops growth and requires EVOLVE. Therefore retaining the current row pointer while incrementing levels inside one GROWTH accept is consistent with the table design.

Current named Aura EXP resources include Fire Runes labelled 10/50/100/250/500, all below the first 1000-EXP row requirement. No current infinite-loop or normal-material arithmetic bug was promoted in this pass.

### EVOLVE material accounting
Normal client requires the checked-in SUB stack itself to meet the configured material count.

Server check-in validates the required material VNUM but not the SUB stack count.

Accept then compares the requirement against `CountSpecifyItem(requiredVnum)` across inventory, while actual success/failure consumption touches only the checked-in `mtrlItem`.

This split between global validation and local consumption enables an undersized checked-in stack to underpay the evolution material requirement when the rest is held in another stack.

See `BUG-AURA-002`.

### Aura Eraser target validation
The Aura Eraser server branch treats `ITEM_SOCKET_AURA_BOOST` as socket index 2 but never verifies the destination is `ITEM_COSTUME / COSTUME_AURA`.

Any valid unequipped/unsealed/non-exchanging target with nonzero socket 2 can therefore reach `SetSocket(2, 0)`.

Booster attachment itself does use `IsAuraBoosterForSocket()`, which explicitly requires COSTUME_AURA, so this gap is specific to the eraser branch.

See `BUG-AURA-003`.

## Current verified findings
- `BUG-AURA-001` — post-open opener-distance gate is bypassed.
- `BUG-AURA-002` — EVOLVE global-count validation / local-stack consumption mismatch.
- `BUG-AURA-003` — Aura Eraser can clear socket2 on non-Aura items.

## Remaining static work
1. audit Aura booster timer/equip/unequip persistence and rate application;
2. close forced-warp/disconnect/destructor locked-item cleanup;
3. audit opener lifetime and cross-window item lifetime;
4. verify wedding-item absorption policy against tracked proto/data;
5. map Aura costume creation/default socket initialization and refine chains;
6. close visual PART_AURA/client asset dependencies;
7. consolidate runtime ownership before STATIC COMPLETE promotion.


## Proto / visual / booster closure pass — 2026-09-28

### Current Aura proto families
Tracked `Project_DumpProto/tr/item_proto.txt` confirms four complete Aura families:
- `49001..49006`;
- `49011..49016`;
- `49021..49026`;
- `49031..49036`.

For every family:
- type/subtype is `ITEM_COSTUME / COSTUME_AURA`;
- grade 1..5 has `RefineSet=409` and `RefinedVnum` pointing to the next grade;
- grade 6 has `RefineSet=409` and terminal `RefinedVnum=0`.

This closes the earlier dataset-dependent EVOLVE-chain question: the server's `GetRefineSet()==409 ? GetRefinedVnum() : GetOriginalVnum()` branch follows the intended current family chain for all tracked Aura costumes.

Creation initialization uses:
`originalVnum % 10 -> grade -> LEVEL_MIN`
and stores the Aura level/EXP encoding in socket 1.

Current tracked Aura costume VNUMs all end in grade digits 1..6, so the grade-0 underflow candidate in the grade-index helper is not reachable from current Aura proto.

### Current Aura resource / booster data
Tracked Aura growth resources:
- 49990: EXP 1;
- 49991: EXP 10;
- 49992: EXP 50;
- 49993: EXP 100;
- 49994: EXP 250;
- 49995: EXP 500.

Tracked booster family:
- 49980: eraser;
- 49981: booster index 1 / 86400 s;
- 49982: index 2 / 259200 s;
- 49983: index 3 / 432000 s;
- 49984: index 5 / unlimited flag 1.

The timed booster lifecycle is internally coherent:
- equip / SetEquipped starts the Aura booster expire event;
- unequip stops it and writes remaining seconds back into socket 2;
- expiry removes the old boosted Aura points, clears socket 2, reapplies Aura points without the booster, recomputes battle points and updates the character packet.

No stale boosted-stat defect was verified.

### Terminal Radiant EVOLVE boundary
Normal client EVOLVE MAIN eligibility rejects `curLevel >= AURA_MAX_LEVEL`.

Server EVOLVE check-in instead accepts a MAIN item when:
`level == table LEVEL_MAX && exp == table NEED_EXP`.

The terminal Radiant row is level 250 / NEED_EXP 0, so a legitimate current grade-6 Aura satisfies the server predicate.

The server then calls `__GetAuraRefineInfo()` for the preview. That helper divides the socket EXP by table NEED_EXP, which is zero for Radiant, and converts the non-finite result to `uint8_t`.

Accept later rejects Radiant, but the arithmetic has already happened at check-in.

See `BUG-AURA-006`.

### Aura visual dependency
Aura rendering is effect-driven rather than GR2/MSM-shape driven.

Canonical client path:
`PART_AURA`
-> `CInstanceBase::SetAura(vnum)`
-> `CItemManager::GetItemDataPointer(vnum)`
-> `CItemData::GetAuraEffectID()`
-> effect attachment on `Bip01 Spine2`.

`item_list.txt` has special four-column `AURA` rows. `CItemManager::LoadItemList()` interprets the fourth column as an Aura `.mse` effect path and registers it through `SetAuraEffectID()`; it is not treated as a GR2 model.

Current mapped examples:
- 49001 -> `aura_01_49_001.mse`;
- 49002 -> `aura_50_99_002.mse`;
- ... through 49006 -> `aura_250_006.mse`;
with equivalent 011/021/031 family variants.

`item_scale.txt` supplies job/sex mesh/particle scale data. For COSTUME_AURA, `CItemManager::LoadItemScale()` propagates a family base row across six consecutive grades.

Future Aura creation therefore requires:
1. compatible COSTUME_AURA proto with grade-ending/refine-chain convention;
2. an `item_list.txt` AURA -> MSE entry for every rendered grade;
3. family scale/particle data in `item_scale.txt` where required;
4. valid MSE/effect assets.

A character MSM ShapeData/GR2 entry is not the primary Aura visual integration path.

### Cross-window / item-type closure
Aura can coexist at open-state level with ChangeLook because neither opener enforces a global window mutex.

However the destructive alias seen in ChangeLook/Acce does not reproduce with current eligibility:
- ChangeLook ITEM mode accepts weapon, ARMOR_BODY or COSTUME_BODY;
- Aura ABSORB material accepts only ARMOR_SHIELD/WRIST/NECK/EAR;
- Aura MAIN accepts COSTUME_AURA;
- Aura GROWTH/EVOLVE SUB accepts Aura resources/evolution materials.

Aura check-in also rejects already locked items and then locks its own stored items.

No distinct Aura/ChangeLook same-item lifetime corruption was proven; the overlap remains a consistency observation.

The client wedding-item filter is redundant for current server ABSORB eligibility: tracked wedding tuxedo/dress/bouquet VNUM classes do not satisfy the server's shield/wrist/neck/ear material predicate.

## Current verified findings
Canonical Aura findings are now:
`BUG-AURA-001..BUG-AURA-006`.

Canonical deferred tests:
`AURA-T01..AURA-T06`.

Remaining closure focus:
- forced server-side warp / disconnect / descriptor-loss lock cleanup;
- opener-null Lua boundary;
- final persistence/check-in lifetime pass;
- readiness consolidation.


## Final lifecycle / persistence closure — 2026-09-28

### Normal close / disconnect
Aura check-in stores inventory positions rather than long-lived item pointers and locks the real checked-in items.

`AuraRefineWindowClose()`:
- clears opener/type/open state and `W_AURA`;
- resolves each stored `TItemPos`;
- unlocks any still-existing item;
- clears every stored Aura slot.

The normal character disconnect sequence calls `AuraRefineWindowClose()` while the descriptor is still bound, before character destruction. Therefore ordinary logout/disconnect does not leave Aura item locks behind.

### Warp boundary
`CanWarp()` and the generic anti-transaction gate both include `W_AURA`, so normal player-controlled warp flows are blocked while Aura is open.

`WarpSet()` itself does not close Aura and can be called directly by trusted server/quest code. A forced direct warp can therefore preserve Aura state. With the currently verified `BUG-AURA-001`, post-open Aura operations do not dereference the opener for distance validation and can continue while the opener pointer merely remains non-null.

No tracked current Aura quest producer or other concrete normal gameplay path was established that combines an open Aura transaction with such a forced warp, so this remains a robustness/integration boundary rather than a separate promoted bug.

### Opener lifetime / null boundary
The Lua bindings pass `GetCurrentNPCCharacterPtr()` directly to `OpenAuraRefineWindow()`; that function immediately reads opener coordinates without a null guard.

No tracked `Project_Game` source call to `game.open_aura_absorb_window`, `game.open_aura_growth_window`, or `game.open_aura_evolve_window` was found in the current snapshot. The tracked object scripts inspected for the conventional blacksmith NPC likewise do not expose Aura entry.

Because a concrete current producer for a null opener was not established, the null-opener dereference remains an integration/content-authoring candidate only.

### Checked-in item lifetime
Current tracked eligible inputs do not expose a normal timer-driven slot replacement path:
- the four Aura costume families have no real-time limit in tracked proto;
- current `RESOURCE_AURA` items have no real-time limit;
- no tracked shield/wrist/neck/ear ABSORB material carries a `LIMIT_REAL_TIME*` limit.

The server also blocks ordinary move/use/drop/destroy/equip flows while Aura is open, and each checked-in item is locked.

No current reachable same-slot replacement/lifetime corruption was promoted.

### Readiness conclusion
All mapped Aura transaction surfaces now have canonical ownership:
- packet/state authorization;
- ABSORB;
- GROWTH;
- EVOLVE;
- booster/eraser;
- Yohara persistence;
- terminal-grade arithmetic;
- packet initialization;
- disconnect/warp state;
- proto/refine chain;
- client visual/effect chain.

Canonical verified findings:
`BUG-AURA-001..BUG-AURA-006`.

Canonical deferred validation:
`AURA-T01..AURA-T06`.

No runtime test has been executed.

**Aura System is STATIC COMPLETE.**
