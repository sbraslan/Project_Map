# Aura System — Static Bug Registry

## BUG-AURA-001 — Aura distance check is structurally bypassed after the window opens

**Status:** VERIFIED STATIC

Server:
- `CHARACTER::IsAuraRefineWindowCanRefine()`
- `AuraRefineWindowCheckIn()`
- `AuraRefineWindowCheckOut()`
- `AuraRefineWindowAccept()`

`IsAuraRefineWindowCanRefine()` starts by calling `CanHandleItem()`.

The generic item gate returns false whenever:
```
IsAuraRefineWindowOpen() || nullptr != GetAuraRefineWindowOpener()
```

Consequently the Aura-specific permission function normally returns false immediately while its own window is legitimately open and never reaches its opener-distance comparison.

Each Aura transaction handler then catches that false result but explicitly continues if:
```
IsAuraRefineWindowOpen() && GetAuraRefineWindowOpener() != nullptr
```

So the fallback condition that is true for a normal open Aura window suppresses the failed permission result.

**Impact:** after opening Aura within the initial allowed distance, the player can move away while retaining the window and the mapped check-in/check-out/final accept paths no longer enforce `AURA_REFINE_MAX_DISTANCE`. Final destructive Aura operations can therefore remain remotely callable as long as the Aura state/opener survives.

**Runtime:** controlled normal-flow distance test only after phase unlock; see `AURA-T01`.


## BUG-AURA-002 — EVOLVE validates total inventory material count but consumes only the checked-in SUB stack

**Status:** VERIFIED STATIC / MODIFIED-CLIENT REACHABLE

Normal client `uiaura.py` requires the attached EVOLVE material stack itself to contain at least the table-required count before sending Aura check-in.

Server check-in does **not** enforce that stack-count invariant. It only requires the SUB item VNUM to match:
`AURA_REFINE_INFO_MATERIAL_VNUM`.

At final `AuraRefineWindowAccept(AURA_WINDOW_TYPE_EVOLVE)`, the server verifies:
`CountSpecifyItem(requiredVnum) >= requiredCount`.

That count is character-wide inventory quantity, not the checked-in `mtrlItem` stack quantity.

Both success and failure consumption then operate only on `mtrlItem`:
- if its stack is larger than the requirement, subtract the requirement;
- otherwise remove that one checked-in item/stack entirely.

Therefore a crafted client can check in an undersized stack while keeping the remaining same-VNUM materials in other stacks. The global count passes, but only the undersized checked-in stack is consumed.

Example for a requirement of 10:
- check in stack count 1;
- keep another count 9 elsewhere;
- total count is 10, so Accept passes;
- consumption removes only the checked-in count-1 stack;
- the remaining 9 survive while the evolution attempt proceeds.

The defect affects both successful and failed evolution attempts.

**Impact:** Aura evolution material cost can be underpaid by splitting the required material across stacks and checking in a deliberately undersized stack.

**Runtime:** Stage B modified-client / disposable-item test only. See `AURA-T02`.

## BUG-AURA-003 — Aura Eraser can clear socket 2 of arbitrary non-Aura items

**Status:** VERIFIED STATIC / DESTRUCTIVE MODIFIED-CLIENT REACHABLE

Server item-use case:
`ITEM_AURA_BOOST_ITEM_VNUM_BASE + ITEM_AURA_BOOST_ERASER`.

The destination checks only:
- valid destination item;
- not exchanging;
- not equipped;
- not sealed;
- destination socket `ITEM_SOCKET_AURA_BOOST` is nonzero.

`ITEM_SOCKET_AURA_BOOST` is enum value **2**, i.e. the ordinary third item socket.

The branch does **not** require:
`ITEM_COSTUME / COSTUME_AURA`.

After consuming the eraser it executes:
`item2->SetSocket(ITEM_SOCKET_AURA_BOOST, 0)`.

Thus a crafted item-use request can target another unequipped item whose generic socket 2 is nonzero and permanently zero that socket. Weapons/armor and other systems can legitimately use ordinary item sockets independently of Aura.

The separate booster-attachment path is stricter because `IsAuraBoosterForSocket()` explicitly requires COSTUME_AURA; the eraser path lacks the equivalent target check.

**Impact:** an Aura-specific consumable can destructively mutate unrelated item socket state.

**Runtime:** Stage B modified-client / disposable-item test only. Never test on valuable or production items. See `AURA-T03`.


## BUG-AURA-004 — Aura SET_ITEM packets expose uninitialized TItemData fields

**Status:** VERIFIED STATIC / PASSIVE INFORMATION DISCLOSURE

Aura check-in/result preview packets are built with:
`TSubPacketGCAuraSetItem sub;`
or
`TSubPacketGCAuraSetItem sub2;`

These objects are not value-initialized or cleared before transmission.

The Aura sender fills only a subset of `TItemData`: VNUM/count/flags/anti-flags, sockets, normal attributes, Yohara random arrays and, in some branches, set value.

In the current build, `TItemData` also contains enabled fields for:
- seal date;
- transmutation VNUM;
- basic-item state;
- element grade/attack/type/value data;
- set value.

The corresponding feature macros are enabled in `CommonDefines.h`.

Because the packet object is sent in full, any member not explicitly assigned by the Aura branch contains indeterminate stack bytes and crosses the server -> client trust boundary.

The client receive path compounds the problem by declaring `TItemData kItemData;` without clearing it and copying only VNUM/count/sockets/attributes/Yohara arrays before storing the Aura slot. In particular, normal Aura tooltip code can request set-value data from this stored object even though the receive path does not initialize/copy it.

**Impact:** Aura window traffic can disclose stale stack bytes in unused metadata fields and can create nondeterministic client-side Aura preview metadata. This is primarily an information-initialization defect; no production exploitation is required to establish it statically.

**Runtime:** Stage C debug packet-capture / sanitizer ownership only. See `AURA-T04`.

## BUG-AURA-005 — Successful Aura evolution drops absorbed Yohara random applies

**Status:** VERIFIED STATIC / DESTRUCTIVE DATA LOSS

With `ENABLE_YOHARA_SYSTEM`, ABSORB success explicitly transfers the source material's Yohara random applies into the Aura:
`mtrlItem->CopyApplyRandomTo(auraItem)`.

On successful EVOLVE the server creates the next Aura item and transfers:
- all sockets through `CopySocketTo()`;
- classic item attributes through `CopyAttributeTo()`.

It does **not** call `CopyApplyRandomTo()` for the new Aura.

The implementations are independent:
- `CopyAttributeTo()` only calls `SetAttributes(m_aAttr)`;
- `CopyApplyRandomTo()` separately calls `SetRandomAttrs(m_aApplyRandom)`.

The old Aura is then removed, so any absorbed Yohara random applies stored on it are lost permanently on a successful grade evolution.

**Impact:** a successfully evolved Aura can silently lose absorbed Yohara random bonus data while its classic absorbed attributes survive.

**Runtime:** Stage B/C disposable-item data-integrity test only. See `AURA-T05`.
