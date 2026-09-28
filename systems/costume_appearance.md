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

## Mapping next

1. close persistence path for transmutation VNUM through DB load/save;
2. inspect ChangeLook clear-scroll/reset behavior;
3. inspect mount appearance consumption/use path;
4. inspect window lifecycle/disconnect cleanup;
5. inspect armor/body ChangeLook compatibility with item automation;
6. promote additional findings only with static evidence.
