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
