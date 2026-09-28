# Costume / Appearance / ChangeLook — Static Map

**Status:** STATIC COMPLETE  
**Phase:** Detection / Mapping Only  
**Source/Game repos:** read-only  
**Opened:** 2026-09-28

## Scope

This subsystem covers:
- ITEM_COSTUME equip/visual behavior;
- body/hair/weapon/acce/aura costume part routing;
- ChangeLook / transmutation UI and server transaction;
- item transmutation VNUM persistence/packet propagation;
- client armor VNUM -> VALUE3 -> MSM ShapeIndex resolution;
- item creation dependencies relevant to costume appearance.

## Costume equip visual flow

### Body costume
Server `game/src/item.cpp`:
- `ITEM_COSTUME / COSTUME_BODY` maps to `PART_MAIN`.
- equipped visual value is the costume item VNUM, or its ChangeLook VNUM when present.

Server sends `PART_MAIN` through character add/update packets.

Client `UserInterface/InstanceBase.cpp`:
`SetArmor(VNUM)`
-> `__ArmorVnumToShape(VNUM)`
-> item data `GetValue(3)`
-> `SetShape(ShapeIndex)`.

Client `GameLib/RaceDataFile.cpp`:
- parses character `.msm` `ShapeData`;
- reads `ShapeIndex`;
- maps ShapeIndex to GR2 model and skin entries.

Therefore canonical body appearance chain is:

`item VNUM -> item_proto VALUE3 -> race/sex MSM ShapeIndex -> GR2/DDS`.

### Hair costume
Server directly resolves `COSTUME_HAIR` to item proto `VALUE3` when equipped. If ChangeLook is present, it reads the material proto's `alValues[3]`.

### Weapon costume
`COSTUME_WEAPON` uses weapon-part routing. Its `VALUE3` has subtype-compatibility semantics elsewhere and must not be treated as a body ShapeIndex field.

### Acce / Aura
- COSTUME_ACCE -> PART_ACCE; ChangeLook VNUM may replace equipped VNUM.
- COSTUME_AURA -> PART_AURA.

## ChangeLook transaction

UI:
`root/uichangelook.py`
-> left target / right material selection
-> client item eligibility helpers
-> `SendChangeLookCheckIn`
-> `SendChangeLookAccept`.

Packet:
- `HEADER_CG_CHANGE_LOOK = 213`
- `HEADER_GC_CHANGE_LOOK = 223`
- client subheaders: CHECK_IN, CHECK_OUT, FREE_ITEM_CHECK_IN, FREE_ITEM_CHECK_OUT, ACCEPT, CANCEL.

Server:
`CInputMain::Transmutation`
-> active `CTransmutation`
-> `ItemCheckIn / ItemCheckOut / Accept`.

Accept:
1. get left target, right material and optional free-ticket;
2. verify target/material presence;
3. verify Yang;
4. deduct price when no free item;
5. `left->SetChangeLookVnum(right->GetVnum())`;
6. update left item packet;
7. clear window slots;
8. destroy right material;
9. destroy free-ticket if used.

Default prices:
- item transmutation: 50,000,000 Yang;
- mount transmutation: 30,000,000 Yang;
- supported free tickets: TRANSMUTATION_TICKET_1 / 2.

## Eligibility

### Item type mode
Left target accepted:
- ITEM_WEAPON except WEAPON_ARROW;
- ITEM_ARMOR / ARMOR_BODY;
- ITEM_COSTUME / COSTUME_BODY.

Right material normally requires:
- different item/VNUM;
- same type;
- same subtype;
- exactly same anti flags.

### Mount mode
Left target accepted:
- quest VNUM 50051, 50052, 50053;
- ITEM_COSTUME / COSTUME_MOUNT.

A special branch exists for 50051..50053 and is currently defective; see BUG-LOOK-002.

## Window/item mutation boundary

`CHARACTER::CanHandleItem()` returns false while ChangeLook window is open.

Normal item movement/drop paths call `CanHandleItem()`, which reduces stale-pointer exposure for items checked into the transmutation window.

The transmutation object itself stores raw `LPITEM` references and does not lock them; any alternate item-mutation path must therefore be reviewed individually before declaring lifetime safety complete.

## ChangeLook item-state propagation

`dwTransmutationVnum` is included in multiple item data paths:
- item set/update;
- safebox/mall;
- guild storage;
- shops;
- exchange;
- equipment/view data where supported.

Client stores inventory transmutation state in `TItemData.dwTransmutationVnum` and updates it through `CPythonPlayer::SetItemTransmutationVnum`.

## Verified findings

- `BUG-LOOK-001` — right-slot check-in before a left target can null-deref in server `CheckOtherItem`.
- `BUG-LOOK-002` — quest-mount targets 50051..50053 accept arbitrary non-costume right materials; the same faulty predicate exists in client and server.
- `BUG-LOOK-003` — CanWarp omits W_CHANGELOOK and WarpSet leaves CTransmutation alive across same-character warp.
- `BUG-LOOK-004` — server accepts sealed/bound right-side ChangeLook material although client UI explicitly rejects it.
- `BUG-LOOK-005` — server allows LEFT checkout/replacement while RIGHT remains, bypassing final type/subtype/anti-flag compatibility.
- `BUG-LOOK-006` — real-time expiry can destroy a checked-in item while CTransmutation retains a dangling raw LPITEM.
- `BUG-LOOK-007` — ChangeLook and Acce windows are not mutually exclusive; Acce can delete an item still retained by ChangeLook.

## Mapping next

1. close mount ChangeLook use/expiry intent and call-chain;
2. inspect hide-costume + ChangeLook interaction;
3. inspect ChangeLook eligibility for weapon/body/job/gender edge cases;
4. inspect alternate item-mutation paths against raw CTransmutation pointers;
5. inspect armor/body ChangeLook compatibility with item automation;
6. promote additional findings only with static evidence.


## Persistence closure — 2026-09-28

ChangeLook VNUM persistence is statically closed end-to-end:

1. `TPlayerItem` contains `dwTransmutationVnum`.
2. `ITEM_MANAGER::SaveSingleItem()` copies `item->GetChangeLookVnum()` into `TPlayerItem`.
3. DB cache `CItemCache::OnFlush()` writes/updates the `item.transmutation` column.
4. Player-item SELECT queries include `transmutation`.
5. `CreateItemTableFromRes()` restores it into `TPlayerItem.dwTransmutationVnum`.
6. Game login item load calls `item->SetChangeLookVnum(p->dwTransmutationVnum)`.
7. Item set/update packets propagate the value to the client.

No persistence defect was verified in this path.

### Clear / reversal scroll
`TRANSMUTATION_CLEAR_SCROLL` / `TRANSMUTATION_REVERSAL`:
- requires a valid target item;
- rejects equipped/exchanging/locked target state;
- requires an existing ChangeLook VNUM;
- calls `item2->SetChangeLookVnum(0)`;
- sends `UpdatePacket()`;
- consumes one scroll.

Because `SetChangeLookVnum()` calls `Save()`, the reset enters the same delayed-save persistence path. No verified clear-scroll persistence bug was found.

## Window lifecycle / warp boundary

`SetTransmutation()`:
- owns/deletes the previous `CTransmutation`;
- sets `W_CHANGELOOK` in the renewed open-window bitset.

Character destruction calls `SetTransmutation(nullptr)`, so disconnect/destruction cleanup is present.

However `CHARACTER::CanWarp()` checks many open-window bits but omits `W_CHANGELOOK`.
`WarpSet()` does not close the active `CTransmutation`.

Therefore a same-character/same-core warp can proceed while the server still owns the ChangeLook object and its checked-in raw item pointers. See `BUG-LOOK-003`.

## Seal/bind validation boundary

Python UI rejects a sealed/bound **right/material** item before sending ChangeLook check-in.

Server `CTransmutation::ItemCheckIn()` and `CheckOtherItem()` validate lock/type/subtype/anti-flags but do not revalidate `IsSealed()`.

The right material is later destroyed by `Accept()`. See `BUG-LOOK-004`.

## Mount ChangeLook expiry observations

`ENABLE_CHANGE_LOOK_MOUNT` is enabled.

Available helper code:
- `StartChangeLookExpireEvent()`;
- `StopChangeLookExpireEvent()`;
- `IsExpireTimeItem()`;
- `GetRealExpireTime()`.

Observed inconsistencies, **not promoted yet**:
- `IsExpireTimeItem()` uses `GetType() != ITEM_COSTUME && GetSubType() != COSTUME_MOUNT`, which is broader than an exact COSTUME_MOUNT filter would be.
- `OnAfterCreatedItem()` starts the ChangeLook expiry event for `IsHorseSummonItem()`, not for COSTUME_MOUNT despite the event helper supporting both.
- the mapped `CTransmutation::Accept()` path sets only the ChangeLook VNUM; it does not visibly initialize socket 2 or call the expiry helper.

These remain deferred until a complete call/intent chain proves live impact.

## Price note

Current live client/server ChangeLook enums agree:
- item transmutation = 50,000,000 Yang;
- mount transmutation = 30,000,000 Yang.

The separate `CL_TRANSMUTATION_PRICE = 15,000,000` common definition appears stale/unused in the mapped transaction and is not a bug by itself.


## Hide-costume + ChangeLook closure — 2026-09-28

The hidden costume getters preserve underlying ChangeLook appearance correctly:

- hidden body costume:
  - `GetPart(PART_MAIN)` resolves the worn body armor;
  - if that armor has `GetChangeLookVnum()`, the transmuted armor VNUM is returned;
  - otherwise the armor VNUM is returned.
- hidden weapon costume:
  - `GetPart(PART_WEAPON)` similarly resolves the base weapon ChangeLook VNUM first.
- hair/acce/aura hidden states intentionally return zero for the hidden visual part.
- toggling hide state calls `UpdatePacket()`, so the dynamic getter is used for the network-visible part.

No verified hide-costume/ChangeLook visual bug was found in the mapped body/weapon path.

## Slot-state invariant audit

Official Python UI prevents removing the left target while the right material remains populated.

Server `CTransmutation::ItemCheckOut()` does not enforce this dependency:
- LEFT can be removed independently;
- RIGHT remains stored.

Server LEFT re-check-in validates only `CanAddItem(newLeft)`.
It does **not** re-run `CheckOtherItem(right)`.

`Accept()` also does not revalidate the final left/right pair.

This makes the material-target relationship mutable after the one-time compatibility check. See `BUG-LOOK-005`.

## Checked-in item lifetime audit

`CTransmutation` stores raw `LPITEM` pointers:
- LEFT;
- RIGHT;
- optional free ticket.

Check-in rejects items that are already locked, but it does not call `Lock(true)`.
The destructor is empty and item destruction does not notify the transmutation object.

Normal player item movement is blocked by `CanHandleItem()` while ChangeLook is open, but asynchronous item lifetime mechanisms remain active.

A concrete verified path is real-time expiry:
1. a time-limited eligible weapon/armor/body-costume is checked into ChangeLook;
2. `real_time_expire_event` reaches its expiry;
3. `ITEM_MANAGER::RemoveItem()` removes the inventory item;
4. `DestroyItem()` erases it and `M2_DELETE(item)`;
5. `CTransmutation::m_Item[]` still contains the old raw pointer.

Subsequent `ItemCheckOut()` or `Accept()` dereferences that stale pointer. See `BUG-LOOK-006`.

## Mount expiry status

Mount ChangeLook expiry remains **deferred / incomplete static intent** rather than promoted:
- `StartChangeLookExpireEvent()`, `StopChangeLookExpireEvent()`, `IsExpireTimeItem()`, and `GetRealExpireTime()` exist;
- `OnAfterCreatedItem()` auto-starts this event only for horse-summon VNUMs 50051..50053 when socket2 is already nonzero;
- `CTransmutation::Accept()` does not initialize socket2 or start this event;
- costume-mount visual resolution itself works through `GetMountVnum() -> transmutation proto APPLY_MOUNT`.

The missing bridge is suspicious but expected product semantics for time-limited source appearance are not yet proven, so it stays unpromoted.

## Eligibility closure notes

Normal item-mode ChangeLook compatibility is checked by:
- same type;
- same subtype;
- exact same anti flags.

That relationship is sound at initial RIGHT check-in, but `BUG-LOOK-005` lets it be bypassed later by changing LEFT.

The additional equip-time ChangeLook checks do not repair this invariant for arbitrary cross-type values generated through that bypass.


## Open-window registry audit — 2026-09-28

The renewed window registry defines:
- W_SAFEBOX
- W_CUBE
- W_EXCHANGE
- W_MYSHOP
- W_SHOP_OWNER
- W_SKILLBOOK_COMB
- W_ATTR_6TH_7TH
- W_ACCE
- W_CHANGELOOK
- W_AURA
- W_SWITCHBOT
- W_GUILDBANK
- W_ROULETTE
- W_MAILBOX

`CTransmutation::Open()` still uses a manual legacy-style list instead of the renewed bitmask and omits at least:
- W_ACCE;
- W_AURA;
- W_GUILDBANK;
- W_ROULETTE;
- W_SWITCHBOT.

Not every omission is independently exploitable, but Acce/Aura are proven interactive overlaps.

### Acce overlap
`OpenAcceCombination()` / `OpenAcceAbsorption()` only guard their own Acce state and can open while ChangeLook is active.
Conversely, `CTransmutation::Open()` does not check Acce state.

ChangeLook check-in does not lock items.
Acce refine/absorption also does not call `CanHandleItem()` before operating on supplied inventory cells.

In absorption mode, a weapon/body-armor item can be used as Acce material and removed by:
`ITEM_MANAGER::RemoveItem(AcceMaterial, "ABSORBED (REFINE SUCCESS)")`.

If that same item is already stored as a ChangeLook LEFT/RIGHT raw pointer, Acce can delete it underneath CTransmutation.
This provides a second concrete trigger for the dangling-pointer condition beyond real-time expiry.

See `BUG-LOOK-007`.

### Aura overlap
`OpenAuraRefineWindow()` does not reject active ChangeLook.
`IsAuraRefineWindowCanRefine()` correctly returns false when ChangeLook is active because `CanHandleItem()` rejects it, but several Aura check-in paths explicitly continue when the Aura window/opener still exist instead of returning.

Aura item slots use `Lock(true)`, so the overlap semantics differ from Acce. It is recorded as a cross-window consistency risk but not promoted separately during this pass.

### Other omitted windows
Guild Storage, Roulette and Switchbot are omitted from `CTransmutation::Open()`.
No additional ChangeLook-specific corruption path was promoted for them in this pass; existing subsystem-specific controls remain authoritative.

## Cross-type client rendering closure

`BUG-LOOK-005` can produce an invalid stored ChangeLook VNUM across allowed ChangeLook classes.

Client body path:
`SetArmor(changeLookVnum)`
-> `__ArmorVnumToShape(vnum)`
-> blindly read item `VALUE3`
-> `SetShape(VALUE3)`.

There is no armor/body type guard inside `__ArmorVnumToShape()`.

Therefore if a weapon VNUM is attached to a body target through BUG-LOOK-005, the weapon's VALUE3 semantics are interpreted as a character ShapeIndex.

Client weapon path:
`SetWeapon(changeLookVnum)`
-> `AttachWeapon(vnum)`
with no item-type compatibility guard before attachment.

Thus BUG-LOOK-005 has a confirmed direct rendering impact: incompatible VNUM semantics reach PART_MAIN/PART_WEAPON interpretation rather than being normalized away client-side.

No independent client crash is proven statically; malformed/missing/wrong visuals remain the expected result class.

## Item-creation companion closure

For future `item ekleyeceğiz` automation, ChangeLook compatibility must be treated as a generation constraint:
- intended transmutation pairs require same TYPE;
- weapon pairs require same weapon SUBTYPE;
- body armor pairs require ARMOR_BODY;
- body costume pairs require COSTUME_BODY;
- normal ChangeLook requires exact ANTI_FLAG equality;
- COSTUME_BODY VALUE3 is ShapeIndex;
- COSTUME_WEAPON VALUE3 remains weapon-subtype semantics;
- body ShapeIndex must exist in every race/sex MSM that is allowed by the generated ANTI_FLAG set;
- if a GR2/MSM mapping is intentionally absent for a race/sex, generated restrictions must prevent that race/sex from equipping the item;
- item_list/icon and item_names records remain companion outputs.


## Static closure — 2026-09-28

**Status:** STATIC COMPLETE  
**Verified bugs:** BUG-LOOK-001..BUG-LOOK-007  
**Deferred tests:** LOOK-T01..LOOK-T07  
**Execution:** LOCKED / NOT RUN

Mapped and closed:
- costume body/hair/weapon/acce/aura visual routing;
- item VNUM -> VALUE3 -> MSM ShapeIndex -> GR2/DDS body path;
- ChangeLook client/server packet and transaction flow;
- persistence/load/save/reversal;
- hide-costume interaction;
- mount appearance resolution;
- slot-state compatibility;
- checked-in raw-item lifetime;
- renewed open-window overlap;
- cross-type client rendering;
- item-creation compatibility constraints.

### Deferred / unpromoted
The following remain observations/candidates and are **not verified bugs**:
- mount ChangeLook expiry helpers are partially disconnected from the mapped Accept path; intended time-transfer semantics are not sufficiently proven;
- `IsExpireTimeItem()` has a broad type/subtype predicate but no proven live caller impact;
- Aura + ChangeLook overlap exists, but a distinct corruption path beyond the documented window inconsistency was not proven;
- GuildStorage/Roulette/Switchbot are omitted from `CTransmutation::Open()`, but no additional ChangeLook-specific corruption was proven;
- free-ticket pointer alias: `m_FreeItem` is not explicitly required to differ from LEFT/RIGHT. Combined with BUG-LOOK-002, a RIGHT/FREE same-pointer double-remove path is plausible **only if** a free-ticket proto is accepted as the mount RIGHT material. The current static evidence did not close the ticket proto type, so this remains a candidate and is not promoted.

No source/game file was modified and no runtime test was executed.
