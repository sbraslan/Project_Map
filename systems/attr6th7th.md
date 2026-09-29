# 6th/7th Attribute — Static Map

**Status:** STATIC MAPPING OPEN / 8 VERIFIED BUGS  
**Phase:** Detection / Mapping Only  
**Opened:** 2026-09-29  
**Source/Game repositories:** READ-ONLY  
**Runtime:** LOCKED / NOT RUN

## Scope
- 6th/7th attribute request/response packet path;
- skillbook combination;
- fragment/support-material transaction ordering;
- NPC_STORAGE delayed-attribute lifecycle;
- client UI and special-inventory integration;
- cross-window and multiplayer item-lifetime boundaries;
- deployed quest entry/retrieval reachability.

## Canonical source snapshots
- Project_ServerSRC: `321712c849aa01d3635b9fe6a9cde79cc0adcb64`
- Project_ClientSrc: `19c1e08141eb8ff2bf22fd8b93df3e4f3a516d81`
- Project_Binary: `c87aea062f8c50e70a854d4e1dc0c75773960e27`
- Project_Game: `4f26f48e0b20b76825a5a6158c992b8c5670f44e`
- Project_DumpProto: `1066f4620af451c5f728260eb5f8a086672e6384`

## Main roots
- `Project_ServerSRC/game/src/Attr6th7th.cpp/.h`
- `Project_ServerSRC/game/src/input_main.cpp::CInputMain::Attr67Send`
- `Project_ServerSRC/game/src/packet.h::TPacketCGAttr67Send`
- `Project_ServerSRC/game/src/questlua_attr6th7th.cpp`
- `Project_ServerSRC/game/src/questlua_game.cpp::game_open_*attr67/skillbook*`
- `Project_ServerSRC/game/src/char.h::ItemAdded/SetOpenSkillBookComb`
- `Project_ServerSRC/game/src/exchange.cpp`
- `Project_ServerSRC/game/src/item.cpp::CItem::SetCount`
- `Project_ServerSRC/game/src/item_manager.cpp::DestroyItem/SaveSingleItem`
- `Project_ClientSrc/UserInterface/PythonNetworkStreamPhaseGame.cpp`
- `Project_ClientSrc/UserInterface/GameType.h`
- `Project_Binary/root/uiattr67add.py`
- `Project_Binary/root/uiskillbookcombination.py`
- `Project_Binary/root/game.py`
- `Project_Game/share/locale/europe/quest/quest_list`

## Packet flow
Client Python UI
→ native `CPythonNetworkStream`
→ `HEADER_CG_ATTR_6TH_7TH`
→ `CInputMain::Attr67Send`
→ subheader:
- CLOSE / OPEN
- SKILLBOOK_COMB
- GET_FRAGMENT
- ADD

Only OPEN/CLOSE mutate `m_isOpenSkillBookComb` / `W_ATTR_6TH_7TH`. Action subheaders execute without proving that OPEN occurred.

## Window model
`CHARACTER::SetOpenSkillBookComb(bool)` also calls `SetOpenedWindow(W_ATTR_6TH_7TH, b)`.

Other server paths include `W_ATTR_6TH_7TH` in cross-window guards alongside exchange/safebox/shop/cube. Therefore the intended design is mutually exclusive window state.

`CInputMain::Attr67Send` does not enforce that state for SKILLBOOK_COMB, GET_FRAGMENT, or ADD.

## Verified bugs

### BUG-ATTR67-001 — action packets bypass OPEN/window authorization
A client can send SKILLBOOK_COMB, GET_FRAGMENT or ADD directly without OPEN. The server therefore executes Attr67 mutations without setting `W_ATTR_6TH_7TH`, bypassing the cross-window exclusion model.

### BUG-ATTR67-002 — duplicate cells reduce the 10-book combination to one book
`CheckCombStart` counts ten packet entries independently and never requires unique cells. Repeating one valid skillbook cell ten times passes the count test. `DeleteCombItems` destroys the item on the first iteration; subsequent reads of the same cell are empty. Reward and gold deduction still occur.

### BUG-ATTR67-003 — Attr67 can invalidate Exchange raw item pointers
Exchange stores offered items as raw `LPITEM` in `m_apItems[]` and marks them exchanging. Attr67 skillbook/material paths do not reject `IsExchanging()`. A crafted Attr67 request can destroy an item still referenced by Exchange. Later Exchange cancellation dereferences `m_apItems[i]` through `SetExchanging(false)`, yielding a dangling-pointer/UAF crash surface.

### BUG-ATTR67-004 — selected Attr67 target is retained as an unsafe raw pointer
GET_FRAGMENT stores the selected target in `CHARACTER::m_AttrItemAdded`. CLOSE does not clear it and item destruction has no mapped back-reference cleanup. Because GET_FRAGMENT can itself bypass OPEN, the target may be destroyed/moved by ordinary item operations. A later ADD reads `get_item->GetID()`, creating a stale-pointer/UAF surface.

### BUG-ATTR67-005 — fragments are consumed before additive validation completes
ADD validates and removes fragments first. It then resolves and validates the additive/support item. Invalid/stale additive input returns after the fragment removal, permanently consuming fragments without starting the Attr67 operation.

### BUG-ATTR67-006 — last skillbook in every configured reward range is unreachable
Skillbook reward uses `std::uniform_real_distribution<>(low, high)` and truncates the real result to `uint32_t`. The distribution interval is `[low, high)`, so the exact upper VNUM is never produced. DumpProto confirms each upper endpoint is a real distinct skillbook.

Affected endpoints include:
`50406, 50421, 50436, 50451, 50466, 50481, 50496, 50511, 50535`.

Context7/cppreference was used only to verify standard-library distribution semantics.

### BUG-ATTR67-007 — tracked deployment has no quest entry/retrieval caller
The feature is compile-enabled on server/client and the Lua APIs are registered:
- `game.open_skillbook_comb`
- `game.open_skillbook_comb_books`
- `game.open_attr67_add`
- Attr67 retrieval bindings.

All 97 quest sources listed by the deployed `quest_list` were scanned and none calls these entry/retrieval APIs. Compiled quest-object naming also contains no Attr67 quest package. The stock tracked deployment therefore has no legitimate quest route into or out of the feature.

This does not neutralize packet-level bugs because action packets remain accepted directly.

### BUG-ATTR67-008 — skillbook special-inventory positions above 255 are truncated in the protocol
With the enabled extended inventory configuration:
- normal inventory: 4 × 45 = 180 slots;
- skillbook special inventory: global slots 180..359.

But `TPacketCGAttr67Send::bCell[10]` stores each skillbook cell as `uint8_t`. Client native code assigns Python global slot integers directly to those bytes. Positions 256..359 therefore wrap/truncate before reaching the server.

## Deployment-shadowed UI parity findings
These are verified source mismatches but are not promoted as separate active gameplay bugs while BUG-ATTR67-007 blocks the stock quest entry:
- skillbook UI displays 1,000,000 Yang while server checks/charges 100,000;
- Attr67 UI permits support-only submission, while server rejects support when fragment count is zero;
- native client subheader range predicates use impossible `< ... && > ...` conjunctions;
- native skillbook packet builder does not bound `PyList_Size` to the fixed ten-cell array.

## Dormant retrieval candidates
- `attr67_get_vnum` dereferences NPC_STORAGE item before its null check.
- `GetItemAttr` stores the empty-slot result in `uint16_t` without independently rejecting -1.
- `GetItemAttr` itself does not enforce elapsed wait time; current intended enforcement appears quest-driven.
No deployed caller was found, so these remain unpromoted pending deployment repair.

## NPC_STORAGE lifecycle
The server persists/restores NPC_STORAGE item rows. Character teardown contains dedicated Attr67 NPC_STORAGE handling. No logout-loss bug is promoted at this stage.

## Current audit cursor
1. Finish private-shop/shop/material cross-window boundary pass.
2. Audit delayed NPC_STORAGE retrieval and reconnect/restart state reconstruction.
3. Audit percent bounds/support item values and rare-attribute mutation.
4. Close remaining client/server parity candidates.
5. Decide STATIC COMPLETE.

Source/Game repositories remain read-only. Runtime remains locked; first future live gate remains `DUNGEON-T09`.
