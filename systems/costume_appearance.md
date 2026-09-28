# Costume / Appearance / ChangeLook — Static Map

**Status:** MAPPING IN PROGRESS  
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
