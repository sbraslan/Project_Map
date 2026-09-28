# Costume / Appearance / ChangeLook — Bug Registry

**Status:** ACTIVE STATIC MAPPING  
**Execution:** NOT RUN

## BUG-LOOK-001 — Right-slot ChangeLook check-in can null-deref when left slot is empty

**Status:** VERIFIED STATIC

Server:
`game/src/Transmutation.cpp -> CTransmutation::CheckOtherItem`

The function obtains:
`const auto target = GetLeftItem();`

It then executes debug/info calls using:
`target->GetVnum()`

**before**:
`if (target == nullptr) return false;`

Packet path:
`HEADER_CG_CHANGE_LOOK`
-> `CInputMain::Transmutation`
-> `ITEM_CHECK_IN`
-> `CTransmutation::ItemCheckIn(pos, slot_type)`
-> right slot
-> `CheckOtherItem(inven_item)`.

`ItemCheckIn` validates slot bounds but does not require the left slot to be populated before invoking the right-slot validator.

Normal Python UI prevents ordinary users from selecting the right slot first, but the server packet handler must still tolerate a validly-sized client request with right `slot_type`.

**Impact:** game-core null dereference / crash candidate from malformed or modified-client ChangeLook sequence.

**Runtime:** deferred. Do not run under current phase.

## BUG-LOOK-002 — Quest-mount ChangeLook accepts arbitrary non-costume material

**Status:** VERIFIED STATIC

Affected special target VNUMs:
- 50051
- 50052
- 50053

Server `CTransmutation::CheckOtherItem` special branch rejects only:
- material type == ITEM_COSTUME
- AND subtype != COSTUME_MOUNT.

It does **not** require:
- material type == ITEM_COSTUME
- AND subtype == COSTUME_MOUNT.

Therefore any non-costume item bypasses the special rejection and reaches `return true`.

Client `CPythonItem::_CheckOtherTransmutationItem` contains the same predicate, and `uichangelook.py` relies on that helper for right-slot eligibility.

Thus this is not only a crafted-packet condition: ordinary UI selection can accept non-costume material for 50051..50053 when the surrounding inventory/UI conditions allow selection.

On Accept:
- left target receives `right->GetVnum()` as its ChangeLook VNUM;
- right material is removed.

**Impact:** unrelated item can be consumed as mount appearance material and an invalid/non-mount VNUM can be persisted as the mount ChangeLook value; downstream mount appearance behavior becomes data-dependent.

**Runtime:** deferred. No live reproduction performed.

## Notes / non-promoted

- Raw LPITEM references in CTransmutation are not themselves locked. Normal MoveItem/DropItem paths are blocked by `CanHandleItem()` while the window is open, so this is not yet promoted as a lifetime bug.
- Exact equality of anti flags for normal item transmutation may be intentional compatibility policy; not a bug.


## BUG-LOOK-003 — ChangeLook open state is omitted from CanWarp

**Status:** VERIFIED STATIC

With `ENABLE_CHECK_WINDOW_RENEWAL`, `CHARACTER::SetTransmutation()` sets:
`W_CHANGELOOK`.

But `CHARACTER::CanWarp()` checks an open-window mask containing exchange, safebox, cube, shop, skillbook/attr, aura and switchbot states while omitting `W_CHANGELOOK`.

`CHARACTER::WarpSet()` also does not call `SetTransmutation(nullptr)`.

Consequences established statically for a same-character warp:
- server permits the warp while the ChangeLook object is active;
- `m_pkTransmutation` remains non-null;
- checked-in `LPITEM` pointers remain owned by that object;
- `CanHandleItem()` continues to reject normal item handling while the object remains active.

The Python ChangeLook window has a local 500-distance auto-close, but this is client-side behavior and does not repair the missing server-side warp/window invariant.

**Impact:** ChangeLook state can survive a server-authorized warp; depending on client phase/UI cleanup this can leave stale transmutation state/raw references and/or item-handling lock until CANCEL or character destruction.

**Runtime:** deferred. Prefer controlled same-core warp observation first; no crafted packet is required if a normal warp path can be invoked while the window is open.

## BUG-LOOK-004 — Server does not reject sealed right-side ChangeLook material

**Status:** VERIFIED STATIC

Feature state:
- server `ENABLE_SEALBIND_SYSTEM` is enabled;
- client ChangeLook UI has an explicit seal check for the **right/material** slot.

Client `root/uichangelook.py` refuses the right item when its seal date is not the default timestamp.

Server `CTransmutation::ItemCheckIn()` checks:
- default inventory position;
- item existence;
- `isLocked()`;
- slot/type compatibility.

Server `CheckOtherItem()` checks type/subtype/anti-flags but does **not** check `IsSealed()`.

A modified client can therefore check in a sealed right material. `Accept()` then uses its VNUM as the new appearance and removes the right item.

**Impact:** server-side policy can consume a sealed/bound item as ChangeLook material despite the official client explicitly forbidding the operation.

**Runtime:** deferred modified-client/isolation test only.

## Deferred observations — not promoted

- Mount expiry helpers appear partially disconnected from the mapped Transmutation Accept path.
- `IsExpireTimeItem()` has an overly broad boolean predicate, but no live caller/impact has yet been closed.
- Normal item movement/drop paths are blocked by `CanHandleItem()` while ChangeLook is active; alternate mutation paths still require audit.


## BUG-LOOK-005 — LEFT replacement bypasses final ChangeLook compatibility

**Status:** VERIFIED STATIC

Normal intended order:
1. LEFT target is checked in;
2. RIGHT material is checked against LEFT by `CheckOtherItem()`;
3. client UI prevents removing LEFT while RIGHT is present;
4. Accept applies RIGHT VNUM to LEFT.

Server does not preserve this state invariant.

`CTransmutation::ItemCheckOut(LEFT)`:
- does not check whether RIGHT is populated;
- simply sets LEFT to nullptr.

A modified client can therefore:
1. check in target A as LEFT;
2. check in compatible material B as RIGHT;
3. check out LEFT only;
4. check in a different target C as LEFT.

The new LEFT path calls only `CanAddItem(C)`.
It does not call `CheckOtherItem(B)`.

`Accept()` checks only that LEFT and RIGHT are non-null and does not revalidate their relationship before:
`left->SetChangeLookVnum(right->GetVnum())`.

Because item-mode `CanAddItem()` independently accepts weapons, body armor and body costumes, this can produce a final pair that would have failed the original same-type/same-subtype/anti-flag checks.

**Impact:** incompatible appearance VNUMs can be attached to otherwise valid target items; resulting client/server visual behavior is data-dependent and can cross normal weapon/armor/costume compatibility boundaries.

**Runtime:** deferred modified-client/isolation test.

## BUG-LOOK-006 — Time expiry can leave dangling ChangeLook item pointers

**Status:** VERIFIED STATIC

`CTransmutation` stores checked-in items as raw `LPITEM` pointers.

Check-in:
- rejects an item if `isLocked()`;
- does not lock the accepted item;
- registers no destruction callback.

Normal manual movement is constrained by `CanHandleItem()`, but item expiry events remain independent.

Concrete path:
1. check in an eligible item that has an active real-time expiry event;
2. keep the ChangeLook window open until expiry;
3. `real_time_expire_event` calls `ITEM_MANAGER::RemoveItem(item, "REAL_TIME_EXPIRE")`;
4. `RemoveItem()` removes it from the character;
5. `DestroyItem()` erases item maps and performs `M2_DELETE(item)`;
6. `CTransmutation::m_Item[]` is not cleared.

Later server operations dereference the stale pointer:
- `ItemCheckOut()` uses `item->GetCell()`;
- `Accept()` uses `left->SetChangeLookVnum(...)` and/or `right->GetVnum()`.

**Impact:** use-after-free / game-core crash candidate reachable through a checked-in time-limited ChangeLook item expiring while the window remains active.

**Runtime:** crash-class test; isolated sanitizer/debug environment only.

## Additional closed observations

- Hide-costume body/weapon getters correctly prefer the underlying armor/weapon ChangeLook VNUM while the costume visual is hidden.
- Initial same-type/subtype/anti-flag compatibility is internally consistent; the vulnerability is the later state transition in BUG-LOOK-005.
