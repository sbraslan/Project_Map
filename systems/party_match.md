# Party Match

**Status:** PARTIAL — ACTIVE
**Phase:** Detection / Mapping Only
**Date:** 2026-09-26

> Source repositories are read-only. Only Project_Map records findings.

## Boundary
Party Match has its own queue/protocol and only hands off to normal `CParty` after a match succeeds.

## Active build
`ENABLE_PARTY_MATCH` is enabled in `common/CommonDefines.h`.

## Server roots
- `game/src/GroupMatchManager.h`
- `game/src/GroupMatchManager.cpp`
- `game/src/input_main.cpp::PartyMatch`
- `game/src/packet.h::TPacketCGPartyMatch/TPacketGCPartyMatch`
- `game/src/char.cpp` logout cleanup
- `game/src/main.cpp` manager construction

## Client roots
- `UserInterface/PythonNetworkStreamPhaseGame.cpp::PartyMatch/RecvPartyMatch`
- `UserInterface/PythonNetworkStreamModule.cpp` Python send bindings
- `UserInterface/PythonNetworkStreamPhaseLoading.cpp::LoadPartyMatchInfo`
- `UserInterface/PythonPlayerModule.cpp::GetPartyMatchInfoMap`
- `root/uipartymatch.py`
- `locale/locale/common/partymatch_info.txt`

## Protocol
CG packet:
- header
- subheader
- signed int map/index

GC packet:
- header
- subheader
- message code
- uint32 index/additional info

Search/cancel are static-size packets.

## Server queue model
`CGroupMatchManager` owns:
`std::unordered_multimap<int, LPCHARACTER> SearchMap`.

`AddSearcher(ch,index)` checks:
- non-null character;
- not already searching;
- nonzero index;
- index exists in the hard-coded `Coordinates()` map;
- minimum level;
- required items;
- character is not already in a normal party.

On success:
- queue entry is added;
- PARTY_MATCH_INFO is returned;
- `CheckPlayers(index)` runs immediately.

## Match completion
Each configured dungeon currently requires two players.

When enough queued players exist:
1. queued players are rechecked for existing parties;
2. required items are rechecked;
3. required items are removed;
4. first character creates a normal `CParty`;
5. remaining characters are joined/linked;
6. success is sent;
7. all matched characters are warped to configured coordinates;
8. queue entries for that index are erased.

Logout explicitly calls `StopSearching`, removing the character from the queue.

## Active configuration
Server `Coordinates()`:
- 351 — level 100 — items 71095x1, 71130x1, 76019x1
- 352 — level 100 — no items
- 353 — level 95 — item 30613x1
- 354 — level 95 — no items
- 356 — level 75 — items 71095x1, 71130x1, 76019x1

The client always loads `locale/common/partymatch_info.txt` through a hard-coded `locale/common` path in `PythonApplication.cpp`.
That active common file matches the server map IDs, level requirements and item requirements.

The extra `locale/country/ae/partymatch_info.txt` file is not used by this loader path and is not treated as an active mismatch.

## Initial findings under audit
- Queue state is process-local; no P2P/DB matchmaking replication has been found yet. Cross-core impact must be established before bug promotion.
- A duplicate SEARCH while already queued causes server `StopSearching(...PARTY_MATCH_HOLD...)`, removing the queue entry. The stock UI normally sends CANCEL rather than duplicate SEARCH, so active reachability is not yet established.
- `AddRestricted/IsRestricted` state currently appears inert: the actual restriction check in `AddSearcher` is commented out.

## Exact next audit
1. Establish whether the process-local SearchMap breaks same-channel matchmaking across game cores.
2. Audit duplicate SEARCH / HOLD client-server state behavior and normal reachability.
3. Audit item consumption -> party creation -> warp failure atomicity.
4. Audit raw LPCHARACTER queue lifetime beyond normal logout.
5. Audit match completion and queue erase under all failure paths.


## Cross-core queue scope closure

The queue is definitively **game-process local**:
- `SearchMap` is a member of the local `CGroupMatchManager` singleton;
- no `HEADER_GG_*`, DB packet, P2P handler or DB table exists for Party Match search state;
- `input_p2p.cpp`, `p2p.cpp` and `common/tables.h` contain no Party Match replication path.

The deployed game topology is multi-core inside the same channel.

Project_Game CH1 contains five distinct processes, all configured with:
`CHANNEL: 1`

Examples:
- core1 — PORT 30003 / P2P 30004
- core2 — PORT 30005 / P2P 30006
- core3 — PORT 30007 / P2P 30008
- core4 — PORT 30009 / P2P 30010
- core5 — PORT 30011 / P2P 30012

Their `MAP_ALLOW` sets split world maps across those processes.

Therefore two players on CH1 searching the same Party Match dungeon from different game cores are inserted into different in-memory `SearchMap` instances. Neither instance can see the other player, so the configured required count can never be satisfied by that pair.

This creates BUG-PMATCH-001.

## Queue lifetime closure
Normal player disconnect executes:
`CHARACTER::Disconnect -> CGroupMatchManager::StopSearching(this, PARTY_MATCH_FAIL, 0)`
before normal party unlink/destruction.

This removes the raw `LPCHARACTER` queue entry on the mapped normal logout path. No additional verified dangling-pointer defect was found.

## Warp atomicity note
`CHARACTER::WarpSet` returns bool and can fail when the target map location cannot be resolved or a private-map relationship is invalid.

Party Match:
- consumes required items;
- creates/joins the party;
- sends success;
- then calls `WarpSet` without checking its return value.

This is a real non-atomic design weakness, but all currently configured Party Match targets have mapped deployment destinations and no normal active failure path has yet been established. It remains documented but unnumbered.

## Verified bugs

### BUG-PMATCH-001 — same-channel matchmaking pool is fragmented per game core
Party Match search state is never replicated outside the local game process, while the active server layout runs multiple game cores under the same channel.

Consequences:
- players on the same channel but different cores cannot satisfy each other's Party Match count;
- queue population is fragmented by the map/core the player happens to be on;
- matchmaking effectiveness depends on source-core population rather than the channel population.

The eventual normal Party created after a match is DB-replicated, but that happens only after a **local** SearchMap reaches the required count and does not solve search-pool fragmentation.

## Exact next audit
1. Close duplicate SEARCH / PARTY_MATCH_HOLD client state desynchronization and reachability.
2. Audit queue behavior when a player changes core/map while searching without a normal disconnect sequence.
3. Audit success notification vs WarpSet return/error handling.
4. Audit item removal semantics and inventory-state restrictions.
5. Close UI/minimap state after every failure/success message.


## Required-item / exchange lifecycle audit

Party Match does not gate SEARCH or match completion on exchange/item-window state:
- `CInputMain::PartyMatch` only routes SEARCH/CANCEL;
- `CGroupMatchManager::AddSearcher` checks index, level, required items and existing party, but not `GetExchange()` or an equivalent window guard;
- `CheckItems(ch,index)` uses `CHARACTER::CountSpecifyItem`;
- `EraseItems(ch,index)` uses `CHARACTER::RemoveSpecifyItem`.

`CountSpecifyItem` skips personal-shop items and sealed items, but does **not** skip `item->IsExchanging()`.

`RemoveSpecifyItem` likewise does not skip exchange-listed items. If the required count consumes the whole item stack, it calls:
`item->SetCount(0)`.

`CItem::SetCount(0)` removes the item from the character and calls `M2_DESTROY_ITEM`, which resolves to `ITEM_MANAGER::DestroyItem`. `DestroyItem` ultimately executes:
`M2_DELETE(item)`.

Exchange keeps the same item as a raw pointer:
- `CExchange::AddItem` stores it in `m_apItems[i]`;
- records its inventory position;
- calls `item->SetExchanging(true)`.

Party Match item deletion does not notify or detach that exchange entry.

Later `CExchange::Cancel` iterates all non-null `m_apItems[i]` and executes:
`m_apItems[i]->SetExchanging(false)`.

Character destruction also explicitly calls `m_pkExchange->Cancel()`.

This creates BUG-PMATCH-002.

### BUG-PMATCH-002 — Party Match can destroy an exchange-listed required item and leave a dangling CExchange pointer
The server accepts Party Match operations while an exchange is active and counts exchange-listed inventory items toward dungeon requirements.

At match completion, an exact required stack can be destroyed while the exchange still stores its raw pointer.

Consequences:
- exchange state retains a pointer to freed `CItem` memory;
- later exchange cancel/destruction dereferences that dangling pointer;
- cross-core Party Match warp makes character destruction/cancel a normal follow-up path for many source-core/target-map combinations;
- even without immediate cross-core destruction, subsequent exchange operations retain invalid item state.

The active Party Match maps 351/352/353/354/356 are hosted primarily on CH1 core3/core5, while searches can originate from other CH1 cores, so a matched character can naturally transition to another core after the item has already been consumed.

This is independent from BUG-PMATCH-001: the first bug fragments search pools; BUG-PMATCH-002 concerns item/exchange lifetime once a local match actually succeeds.

## Exact next audit
1. Close duplicate SEARCH / PARTY_MATCH_HOLD client-state desynchronization and reachability.
2. Audit other conflicting windows/states (safebox, refine, shop, change-look) against required-item consumption.
3. Audit success notification vs WarpSet failure/return ordering.
4. Audit UI/minimap state for every Party Match result code.
5. Decide whether additional queue/core transition bugs remain.
