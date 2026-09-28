# Acce / Sash — Bug Registry

**Status:** STATIC MAPPING IN PROGRESS  
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

No runtime reproduction has been performed.
