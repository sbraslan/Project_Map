# Acce / Sash — Static Map

**Status:** STATIC COMPLETE  
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
- `BUG-ACCE-005` — reversal clears absorbed attributes after the only target update packet, leaving stale client attribute/tool-tip state.
- `BUG-ACCE-006` — absorption does not require an empty target sash; a crafted request can overwrite an existing absorbed item/state and consume the new material.
- `BUG-ACCE-007` — Acce open state can survive warp and keep item handling blocked.
- `BUG-ACCE-008` — reversal never clears copied element/set metadata, so stale extended state is persisted and can remain visible after refresh/relog.

## Historical mapping checklist — CLOSED

The originally listed remaining passes are now closed:
- absorbed-stat math and persistence;
- client model/scale dependencies;
- window/warp/item-mutation lifecycle;
- combine/refine-chain and output placement;
- extended reversal metadata;
- deferred runtime ownership.

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


## Client visual / asset boundary — 2026-09-28

The equipped sash visual is VNUM-driven, not a per-sash character MSM shape entry.

Static path:
1. server exposes the equipped Acce VNUM through `PART_ACCE` (or ChangeLook VNUM when present);
2. client `CInstanceBase::SetAcce(eAcce)` calls `CActorInstance::AttachAcce(eAcce, ..., PART_ACCE)`;
3. `CItemManager` resolves the VNUM's `CItemData`;
4. `item_list.txt` rows with four fields provide icon + GR2 model path through `SetDefaultItemData`;
5. `AttachAcce` uses the item model (sub-model fallback -> model) and attaches it to `Bip01 Spine2`.

Current asset examples are registered in `locale/locale/common/item_list.txt` as `WING` rows, including the 850xx and 860xx Acce ranges, with direct `d:/ymir work/item/wing/...gr2` paths.

This matters for future item creation: a new sash visual needs a valid client item-list/model mapping in addition to a compatible COSTUME_ACCE proto/refine definition. It is not sufficient to add only a generic character MSM ShapeData record.

## Window / warp lifecycle audit

Server-side item mutation through normal handlers is blocked while:
`m_bAcceCombination || m_bAcceAbsorption`
because `CHARACTER::CanHandleItem()` returns false.

However:
- `OpenAcceCombination()` / `OpenAcceAbsorption()` only guard against another Acce mode already being open;
- the final Acce transaction bypasses `CanHandleItem()`;
- `CanWarp()` omits `W_ACCE` from its opened-window mask;
- `CHARACTER::IsHack()` transaction-window masks also omit `W_ACCE`;
- the only server call site found for `AcceClose()` is the client close request handler.

The normal client mitigates this lifecycle gap by:
- calling `AcceWindow.Close()` when the character moves more than 500 units from the open position;
- sending `SendAcceRefineCanCle()` -> server close request.

Because the close is client-driven and no server item pointers are retained by the Acce UI, this is recorded as a lifecycle integration gap rather than a separate verified destructive bug at this checkpoint. It reinforces BUG-ACCE-001: authoritative transaction state is not server-bound.

## Additional absorption invariant

Normal `uiacce.py` only accepts the left absorption target when:
`GetItemMetinSocket(attachedSlotPos, 0) == 0`.

Server `AcceRefine(..., bAcceWindow == 1, ...)` does not check socket 0 before:
- replacing socket 0 with the new material VNUM;
- replacing copied attributes/element/random/set metadata;
- deleting the new material.

Therefore an occupied/previously absorbed sash can be re-absorbed through a crafted final request. See `BUG-ACCE-006`.


## Re-absorption policy gap — 2026-09-28

Official client absorption LEFT eligibility requires an empty absorbed-item socket:
`GetItemMetinSocket(attachedSlotPos, 0) == 0`.

Server `CHARACTER::AcceRefine(..., bAcceWindow == 1)` does not enforce the same invariant.

After type/apply/material checks it directly overwrites:
`AcceItem->SetSocket(0, AcceMaterial->GetVnum())`
and copies normal/element/random/set metadata before consuming the new source item.

Therefore a modified client can use an already-absorbed sash as the target and replace its absorption directly, without first using reversal item 39046/90000.

This is a server/client policy mismatch and bypasses the intended reversal step/cost. See `BUG-ACCE-006`.

## Warp / Acce-window lifecycle

Server open state:
- `OpenAcceCombination()` -> `m_bAcceCombination = true`, `W_ACCE = true`;
- `OpenAcceAbsorption()` -> `m_bAcceAbsorption = true`, `W_ACCE = true`;
- `AcceClose()` is the explicit clear path.

`CanHandleItem()` rejects item handling while either Acce boolean is true.

But `CHARACTER::CanWarp()` does not include `W_ACCE` in its open-window mask and `WarpSet()` does not call `AcceClose()`.

Client `uiAcce.AcceWindow.Close()` does send `HEADER_CG_ACCE_CLOSE_REQUEST`, and its distance watcher closes the dialog when the player moves more than 500 units from the opening point. However the client interface does not provide a separate guaranteed Acce close in its generic warp/loading teardown path; `wndAcce.Close()` appears only in the explicit dialog/refine conflict path.

Thus server correctness currently depends on a client CLOSE packet rather than enforcing the warp invariant itself. A warp can preserve Acce open state and leave `CanHandleItem()` blocked after arrival if CLOSE is absent/lost during transition.

See `BUG-ACCE-007`.

## EFFECT_ACCE_BACK closure

Client `SetAcce()` contains:
`if (m_acceRefineEffect) m_acceRefineEffect = __AttachEffect(EFFECT_ACCE_BACK)`.

Taken alone this looked like a reversed initial-attach condition because the handle initializes to zero.

The full flow closes that concern:
- server equip path emits `SE_ACCE_BACK` when sash socket1 > 18;
- the client special-effect path maps it to `EFFECT_ACCE_BACK`;
- `AttachSpecialEffect(EFFECT_ACCE_BACK)` assigns `m_acceRefineEffect`;
- the effect itself is registered in `playersettingmodule.py` as `D:/ymir work/pc/common/effect/armor/acc_01.mse`.

Therefore `SetAcce()` is not the only/initial producer of this effect handle. No standalone missing-effect bug is promoted from that condition.


## Final static closure — 2026-09-28

### Combine / refine-chain
Final combine audit found no additional promoted defect beyond `BUG-ACCE-004`.

For legitimate distinct inputs:
- same drain-grade inputs are required;
- price is derived from the primary/left sash;
- success follows the primary sash's `GetRefinedVnum()` chain;
- terminal-grade output can raise socket1 up to 25;
- primary absorbed state/attributes/extended metadata are preserved into the result;
- primary and material are removed and output returns to the primary cell;
- failure consumes only material.

Different sash families therefore intentionally follow the left/primary refine chain. No separate cross-family output bug was verified.

The final request uses uint8 inventory cells, but the normal default inventory is 4 pages x 45 = 180 cells, so no normal Acce slot truncation boundary exists above 255 for the supported costume/weapon/body-armor inputs.

### Client model and scale dependency
Acce visual data is not driven by a generic character MSM ShapeData entry.

Canonical current path:
`PART_ACCE`
-> `CInstanceBase::SetAcce()`
-> `CActorInstance::AttachAcce()`
-> VNUM `CItemData`
-> `item_list.txt` WING model
-> attach to `Bip01 Spine2`.

`locale/locale/common/item_list.txt` contains direct WING/GR2 mappings for current 850xx/860xx sash families.

`locale/locale/common/item_scale.txt` provides per-job/per-sex scale rows for those families. `CItemManager::LoadItemScale()` loads these rows and applies each base row across the configured grade range through `SetItemTableScaleData()`.

For future sash creation, the visual dependency is therefore:
1. compatible COSTUME_ACCE item/refine proto;
2. `item_list.txt` WING -> GR2 mapping;
3. `item_scale.txt` job/sex scale data where required;
4. valid GR2 asset.

A character MSM shape entry alone is not the current sash integration path.

### Cross-window ownership
`OpenAcceCombination()` and `OpenAcceAbsorption()` only prevent a second Acce mode; they do not enforce a global opened-window mutex.

This architecture already has one concrete cross-system destructive finding owned by:
`BUG-LOOK-007` — Acce overlap with ChangeLook can invalidate a ChangeLook-retained raw item pointer.

Aura can likewise coexist at the open-state level because neither opener globally excludes the other, but no new Acce-specific corruption/lifetime consequence was proven in this pass. It remains an integration observation, not a duplicate bug ID.

### Reversal extended metadata
The earlier observation that reversal leaves element/random/set metadata is promoted to `BUG-ACCE-008` after closing the persistence and client-consumer chain.

Socket0 correctly gates the live absorbed gameplay bonuses, but:
- copied element/set metadata is never cleared;
- it is included in normal item persistence;
- generic tooltip/title code can consume it independently of socket0.

Thus the stale state can survive a full item refresh and relog.

## Closure result

**Acce / Sash is STATIC COMPLETE.**

Canonical verified findings:
`BUG-ACCE-001..008`.

Canonical deferred tests:
`ACCE-T01..ACCE-T08`.

No runtime test has been executed. Runtime order remains locked with `DUNGEON-T09` first.
