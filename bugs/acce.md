# Acce / Sash — Bug Registry

**Status:** STATIC COMPLETE  
**Execution:** LOCKED / NOT RUN

## BUG-ACCE-001 — Final refine packet is not gated by active server Acce window/mode

**Status:** VERIFIED STATIC

Client check-in/check-out is local bookkeeping only. The only transaction packet is the final:
`HEADER_CG_ACCE_REFINE_REQUEST`.

Server path:
`CInputMain::AcceRefineRequest`
-> `CHARACTER::AcceRefine(packet mode, primary cell, material cell)`.

`AcceRefine()` does not require:
- `m_bAcceCombination == true` for combine;
- `m_bAcceAbsorption == true` for absorb;
- `W_ACCE` to be open;
- requested packet mode to match the server-side mode opened by the quest/UI flow.

Thus the final operation is packet-driven rather than bound to the authoritative open-window state.

**Impact:** a modified client can reach the Acce transaction logic without using the intended quest/window lifecycle, including in states where the normal UI would not expose the action.

**Runtime:** deferred. No packet crafting under current phase.

---

## BUG-ACCE-002 — Incorrect AND predicate permits wrong costume subtypes as sash inputs

**Status:** VERIFIED STATIC

Combine validation uses:

```cpp
if ((AcceItem->GetType() != ITEM_COSTUME && AcceItem->GetSubType() != COSTUME_ACCE) ||
    (AcceMaterial->GetType() != ITEM_COSTUME && AcceMaterial->GetSubType() != COSTUME_ACCE))
    return false;
```

Absorb-left validation uses the same logical pattern for the target sash.

For an item whose type is `ITEM_COSTUME` but subtype is not `COSTUME_ACCE`:
- `GetType() != ITEM_COSTUME` is false;
- `GetSubType() != COSTUME_ACCE` is true;
- `false && true` is false;
- therefore this predicate does not reject the wrong subtype.

Normal Python UI requires both exact type and exact Acce subtype, so client and server policies differ.

Other later conditions can still restrict concrete protos, especially `APPLY_ACCEDRAIN_RATE`; the defect is the server's missing exact sash-type invariant.

**Impact:** wrong costume classes can reach Acce transaction logic whenever their remaining apply/state conditions satisfy the later checks.

**Runtime:** deferred.

---

## BUG-ACCE-003 — Absorption accepts non-body armor because ARMOR_BODY is compared as an item type

**Status:** VERIFIED STATIC

Server absorption material check:

```cpp
if (AcceMaterial->GetType() != ITEM_WEAPON &&
    AcceMaterial->GetType() != ITEM_ARMOR &&
    AcceMaterial->GetType() != ARMOR_BODY)
    return false;
```

Canonical constants:
- `ITEM_WEAPON = 1`
- `ITEM_ARMOR = 2`
- `ARMOR_BODY = 0` — this is an armor **subtype**, not an item type.

Because every armor subtype still has `GetType() == ITEM_ARMOR`, the second comparison is false and the item passes regardless of armor subtype.

The normal client UI permits:
- weapon;
- armor only when subtype == `ARMOR_BODY`.

Therefore server validation is broader than client validation.

**Impact:** a modified client can submit helmet/shield/wrist/foot/neck/ear/etc. armor as Acce absorption material. The material can be consumed and its VNUM/attributes written into the sash state even though the UI does not permit that class.

**Runtime:** deferred.

---

## BUG-ACCE-004 — Same inventory cell can be used as both combine inputs

**Status:** VERIFIED STATIC

`AcceRefine()` independently resolves:
- `GetInventoryItem(bSlotAcce)`
- `GetInventoryItem(bSlotMaterial)`

There is no server check requiring:
`bSlotAcce != bSlotMaterial`
or
`AcceItem != AcceMaterial`.

A crafted final combine request can therefore alias both pointers to one item.

### Failure branch
The server executes:
`RemoveItem(AcceMaterial, "COMBINE (REFINE FAIL)")`.

Because material == primary, the primary sash itself is consumed.

### Success branch
The server executes:
1. `RemoveItem(AcceItem, ...)`;
2. `RemoveItem(AcceMaterial, ...)`.

The first removal enters the established `RemoveItem -> DestroyItem -> M2_DELETE` lifecycle. The second call therefore receives the stale alias of the already-destroyed object.

**Impact:** destructive same-slot transaction; failure can delete the primary sash, while success reaches a use-after-free/double-remove crash-corruption candidate.

Normal client slot bookkeeping attempts to prevent duplicate registration, but server final packet validation does not.

**Runtime:** only in isolated sanitizer/debug environment if this phase is later authorized. Never production.

## Current coverage

Verified static bugs:
- BUG-ACCE-001
- BUG-ACCE-002
- BUG-ACCE-003
- BUG-ACCE-004
- BUG-ACCE-005
- BUG-ACCE-006
- BUG-ACCE-007
- BUG-ACCE-008

No runtime reproduction has been performed.


---

## BUG-ACCE-005 — Reversal sends the target update before clearing absorbed attributes

**Status:** VERIFIED STATIC

Reversal path:
`char_item.cpp`
1. `item2->SetSocket(0, 0)`;
2. `item2->ClearAllAttribute()`;
3. consume reversal scroll.

`SetSocket()` immediately calls:
- `UpdatePacket()`;
- `Save()`.

The item update packet contains the full socket and attribute arrays. At that moment the old absorbed attributes are still present.

`ClearAllAttribute()` then zeroes the server-side attribute array but performs neither:
- `UpdatePacket()`;
- nor its own `Save()`.

There is no later target-sash `UpdatePacket()` in the reversal branch.

Client Acce tooltip `__AppendAttributeInformationAcce()` reads the client-side attribute slots and does not require absorbed socket0 to be nonzero before rendering them. It derives drain percentage from socket1.

Therefore immediately after reversal:
- server target sash has socket0=0 and cleared normal attributes;
- client target sash has socket0=0 but retains the old attribute array from the earlier packet;
- tooltip can continue displaying the removed absorbed bonuses until another full item refresh occurs.

The earlier `SetSocket()->Save()` is a delayed save; when flushed it serializes the current item object, including the cleared attributes. Thus no separate DB-persistence defect is established here.

**Impact:** verified client/server item-state and tooltip desynchronization after Acce reversal; displayed bonuses can be stale even though the server has removed them.

**Runtime:** deferred; ordinary UI observation is sufficient if later authorized.


---

## BUG-ACCE-006 — Absorption can overwrite an already-occupied sash

**Status:** VERIFIED STATIC

Normal client left-target admission in `uiacce.py` requires:
```python
if playerm2g2.GetItemMetinSocket(attachedSlotPos, 0) == 0:
    possablecheckin = 1
```

So the intended client workflow rejects a sash that already contains an absorbed source VNUM in socket 0.

Server absorption path validates the target's broad costume/apply identity but never requires:
`AcceItem->GetSocket(0) == 0`.

It then immediately:
1. `SetSocket(0, AcceMaterial->GetVnum())`;
2. overwrites copied normal attributes;
3. overwrites enabled element/random/set metadata;
4. `RemoveItem(AcceMaterial, "ABSORBED (REFINE SUCCESS)")`.

Therefore a crafted final absorb request can submit a previously absorbed sash as the target.

**Impact:** the prior absorbed source/state is overwritten without the reversal workflow, while the newly supplied weapon/armor is consumed. This bypasses the normal client invariant that absorption is only performed into an empty sash.

This finding is independent of BUG-ACCE-003: even a normally valid weapon or body-armor material can trigger the overwrite if the target is already occupied.

**Runtime:** deferred; use disposable items only if later authorized.


---

## BUG-ACCE-007 — Acce open state can survive warp and keep item handling blocked

**Status:** VERIFIED STATIC

Server Acce open state is represented by:
- `m_bAcceCombination`;
- `m_bAcceAbsorption`;
- and, with renewed window tracking, `W_ACCE`.

`CHARACTER::CanHandleItem()` rejects normal item operations while either Acce boolean is true.

However:
- `CHARACTER::CanWarp()` does not include `W_ACCE` in its opened-window mask;
- the mapped `IsHack()` transaction-window masks also omit `W_ACCE`;
- `WarpSet()` does not call `AcceClose()`;
- the mapped server close path is the client Acce CLOSE request.

The normal client mitigates this by distance-closing the Acce UI and sending CLOSE, but server correctness still depends on that client packet arriving before/during transition.

**Impact:** after a server-authorized warp, stale Acce flags can survive and continue making `CanHandleItem()` reject normal inventory/item operations until Acce state is explicitly cleared or the session resets.

**Runtime:** deferred ordinary/controlled warp-state observation.


---

## BUG-ACCE-008 — Reversal leaves copied element/set metadata persistently attached to the sash

**Status:** VERIFIED STATIC

Absorption copies extended source metadata when the corresponding features are enabled, including:
- element grade/type/value data;
- Yohara/random item apply data;
- `set_value`.

The reversal item path clears only:
1. absorbed source socket 0;
2. normal item attributes.

It does **not** clear the copied element/random/set fields.

Gameplay application of the absorbed Acce bonuses is largely protected by socket0:
- Acce absorbed weapon/armor point application requires a nonzero absorbed source;
- copied Yohara/random Acce application is also socket0-gated;
- the sash slot is not part of the character's normal equipment set-count slots.

But the client has persistent UI consumers that are not socket0-gated:
- `AddItemData(..., set_value)` passes `set_value` into the item-title path, where `SetItemString(set_value)` can prepend a set label;
- the generic element tooltip path can render a positive stored element grade before the Acce-specific socket0-gated branch.

Persistence is explicit:
- `SaveSingleItem()` copies `set_value`, element fields and enabled random fields into `TPlayerItem`;
- DB cache/player load persists and restores them;
- reversal's `SetSocket(0,0)` schedules save while those extended fields are unchanged.

**Impact:** after reversal, absorbed gameplay stats stop because socket0 is zero, but copied extended metadata can remain durably attached to the sash and remain visible in tooltip/title even after full refresh or relog. This is a persistent inconsistent item state.

This is distinct from `BUG-ACCE-005`:
- 005 is transient stale **normal attributes** caused by packet ordering; delayed save eventually stores them cleared;
- 008 is **extended metadata never cleared at all**, so DB persistence preserves it.

**Runtime:** deferred UI/persistence observation using disposable metadata-bearing items.
