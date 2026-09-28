# Aura System — Static Map

**Status:** STATIC MAPPING IN PROGRESS  
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
