# Acce / Sash — Static Map

**Status:** STATIC MAPPING IN PROGRESS  
**Phase:** Detection / Mapping Only  
**Source/Game repos:** read-only  
**Opened:** 2026-09-28  
**Execution:** LOCKED / NOT RUN

## Scope

This subsystem covers:
- Acce / shoulder sash combination and absorption;
- client Acce slot bookkeeping and packet boundary;
- server-side refine/absorb validation;
- sash absorbed-item storage and stat application;
- sash equip/visual routing;
- ChangeLook interaction relevant to Acce;
- reversal/reset behavior and remaining persistence/data boundaries.

## Entry / open flow

Quest entry is exposed through:
- `pc.open_acce_absorption`
- `pc.open_acce_combination`

Both call the corresponding `CHARACTER::OpenAcce*` function.

Server open state:
- `OpenAcceCombination()` sets `m_bAcceCombination = true`;
- `OpenAcceAbsorption()` sets `m_bAcceAbsorption = true`;
- with window renewal enabled both set `W_ACCE`;
- `AcceClose()` clears both booleans and `W_ACCE`.

## Client transaction model

The Acce UI keeps selected inventory cells client-side.

`netSendAcceRefineCheckIn(...)` parses the attached inventory/window/slot values, but its network send is commented out.

`netSendAcceRefineCheckOut(...)` only clears `CPythonPlayer::m_iAcceActivedItemSlot[]`.

Only the final accept transmits the selected cells:

`uiacce.py::Accept`
-> `netSendAcceRefineAccept(type)`
-> read local active slot 0/1
-> `CPythonNetworkStream::SendAcceRefinePacket`
-> `HEADER_CG_ACCE_REFINE_REQUEST`.

Therefore the server receives no authoritative prior check-in state and the final packet handler must validate the complete transaction independently.

## Server final transaction

`CInputMain::AcceRefineRequest`
-> `CHARACTER::AcceRefine(bAcceWindow, bSlotAcce, bSlotMaterial)`.

The request contains:
- requested mode;
- primary inventory cell;
- material inventory cell.

### Combine mode — bAcceWindow == 0
Server:
- resolves both cells through `GetInventoryItem`;
- rejects equipped, locked and sealed items;
- checks sash apply data and grade;
- checks Yang;
- rolls success;
- on success creates refined sash, copies relevant item state, removes both inputs and inserts output;
- on failure consumes the material and deducts Yang.

### Absorb mode — bAcceWindow == 1
Server:
- resolves sash/material cells;
- validates sash/material partially;
- writes material VNUM into sash socket 0;
- copies attributes/element/random/set data;
- removes the absorbed material.

## Client/server validation mismatch

The normal Python UI is stricter than the server in several important places:
- client requires ITEM_COSTUME + COSTUME_ACCE for sash inputs;
- client absorb material allows weapon or ARMOR_BODY only;
- client slot bookkeeping attempts to prevent duplicate inventory-cell registration;
- client workflow assumes an opened Acce window.

The server final packet cannot rely on these client checks because check-in is client-local only.

See BUG-ACCE-001..004.

## Absorbed-item and equip model

Server item behavior:
- `ITEM_COSTUME / COSTUME_ACCE` maps to `WEAR_COSTUME_ACCE`;
- equipped visual routes through `PART_ACCE`;
- ChangeLook VNUM can replace the visible sash VNUM;
- absorbed source VNUM is stored in socket 0;
- drain/refine percentage state is stored in socket 1 by the mapped combination path;
- `CItem::ModifyPoints` reads the absorbed item table and applies drained armor/weapon stats and attributes.

Acce reversal items clear socket 0 and clear attributes on the target sash.

## Verified findings

- `BUG-ACCE-001` — final Acce refine/absorb packet is not bound to an active server-side Acce window or matching server mode.
- `BUG-ACCE-002` — sash type/subtype checks use an incorrect AND predicate, permitting wrong ITEM_COSTUME subtypes when other conditions line up.
- `BUG-ACCE-003` — absorption material validation compares item type against `ARMOR_BODY` and therefore accepts every ITEM_ARMOR subtype rather than body armor only.
- `BUG-ACCE-004` — combine accepts identical primary/material inventory cells; failure can consume the primary sash and success reaches a stale-pointer/double-remove path.

## Mapping next

1. close absorbed-stat math and special apply behavior;
2. close save/load persistence of sash sockets/attributes and reset flow;
3. inspect Acce-specific proto/data ranges and client visual definitions;
4. inspect open-window / warp / item-mutation lifecycle beyond the already-mapped ChangeLook overlap;
5. inspect combine grade/refine-chain edge cases and output placement;
6. promote additional findings only with static evidence.

No source/game file was modified and no runtime test was executed.


## Persistence / stat closure — 2026-09-28

Acce primary persistent state is carried by the normal item record:
- socket 0 = absorbed source VNUM;
- socket 1 = absorption percentage / final-grade drain value;
- normal item attributes = copied absorbed attributes;
- with enabled extensions, element/random/set fields are also copied by the absorb/combine paths.

Persistence chain is closed:
1. `CItem::SetSocket()` updates packet state and schedules `Save()`;
2. `SaveSingleItem()` copies all sockets and normal attributes into `TPlayerItem`;
3. DB cache writes socket/attribute columns;
4. item SELECT/load restores them through `CreateItemTableFromRes()`;
5. game item load restores the item fields.

No Acce-specific socket/normal-attribute persistence defect was verified in this pass.

### Drain math
`CItem::GetDrainPercentage()` clamps socket 1 to 1..25.
`DrainedValue(v)` returns `floor(v * drainPct / 100)`.

Creation initializes an Acce item's socket 1 from `APPLY_ACCEDRAIN_RATE`; grade-4 apply value 20 is randomized to 11..19, while final-grade combination can raise socket 1 up to 25.

Equipped Acce stat application is gated by COSTUME_ACCE plus nonzero absorbed socket 0. Body armor gets drained defense and eligible positive proto applies; weapons get drained physical/magic attack and eligible positive proto applies; copied normal/random attributes are also drained before application.

## Reversal / reset audit

Reversal scroll VNUMs 39046 and 90000:
- require a valid unequipped, unlocked, unsealed COSTUME_ACCE target;
- execute `SetSocket(0, 0)`;
- then execute `ClearAllAttribute()`;
- consume one scroll.

`ClearAllAttribute()` itself neither calls `UpdatePacket()` nor `Save()`.

Because `SetSocket(0,0)` sends its full item update **before** attributes are cleared, the client receives socket0=0 together with the old attribute array. No later target-item update is sent in this reversal branch.

The server's delayed-save pointer will later serialize the now-cleared normal attributes, so this is not promoted as a DB persistence loss. It is, however, a verified client/server item-state desynchronization. See `BUG-ACCE-005`.

The reversal branch also does not explicitly clear copied element/random/set metadata. Their direct gameplay effect is currently gated or outside the Acce wear-set count mapped here, so this remains an observation rather than a separate promoted bug.
