# Party Match — Bug Registry

**Phase:** Detection / Mapping Only

Verified Party Match bugs are listed below.

## Candidates under closure
- process-local queue vs multi-core/channel architecture;
- duplicate SEARCH removes queue entry while HOLD leaves stock client state unchanged;
- item consumption before warp completion;
- raw `LPCHARACTER` queue lifetime.

Candidates remain unnumbered until active reachability/impact is statically closed.


### BUG-PMATCH-001 — same-channel matchmaking is split into independent per-core queues
- Statik durum: **doğrulandı**
- Sınıf: multi-core architecture / matchmaking reachability

`CGroupMatchManager::SearchMap` exists only inside each game process.

There is no Party Match search-state replication through:
- game P2P/GG packets;
- DB GD/DG packets;
- shared DB state.

The active Project_Game topology proves that a single channel is multi-core: CH1 has core1..core5 and every one declares `CHANNEL: 1` with distinct game/P2P ports and different `MAP_ALLOW` sets.

Thus:
- player A on CH1/core1 and player B on CH1/core3 may search the same dungeon;
- A exists only in core1's SearchMap;
- B exists only in core3's SearchMap;
- neither local manager reaches the required count from those two users.

The post-match `CParty` object is DB/channel replicated, but queue state is not; replication begins too late to fix matchmaking.

This is an active architectural defect for the mapped deployment, not merely a theoretical multi-core concern.


### BUG-PMATCH-002 — required Party Match item can be destroyed while still referenced by CExchange
- Statik durum: **doğrulandı**
- Sınıf: item lifecycle / dangling pointer / use-after-free risk

Verified chain:
1. `CExchange::AddItem` keeps a raw `LPITEM` in `m_apItems[]` and sets `IsExchanging=true`.
2. Party Match SEARCH has no exchange-state rejection.
3. `CheckItems` uses `CountSpecifyItem`, which counts exchange-listed items.
4. Match completion calls `RemoveSpecifyItem`, which does not exclude `IsExchanging` items.
5. Consuming an exact stack reaches `CItem::SetCount(0)`.
6. `SetCount(0)` calls `ITEM_MANAGER::DestroyItem`.
7. `DestroyItem` ends with `M2_DELETE(item)`.
8. `CExchange::m_apItems[]` is not cleared by this removal path.
9. `CExchange::Cancel` later calls `m_apItems[i]->SetExchanging(false)` on the stale pointer.
10. `CHARACTER::Destroy` automatically calls `m_pkExchange->Cancel()`, giving a normal follow-up dereference path, especially when Party Match warps the player to another game core.

Partial-stack consumption can also leave exchange-visible state inconsistent even when the object survives; exact-stack consumption is the critical freed-pointer case.

The server must not assume the stock UI prevents this: `HEADER_CG_PARTY_MATCH` is accepted while exchange state is active.


### BUG-PMATCH-003 — FAIL_NO_ITEM removes server queue state but leaves minimap matchmaking icon visible
- Statik durum: **doğrulandı**
- Sınıf: client/server UI state divergence

After a successful search:
- server has the player in SearchMap;
- client `__SetInfo` sets SEARCHING and shows the minimap Party Match button.

If the queued player no longer has a required item when a later `CheckPlayers` occurs, server:
`StopSearching(player, PARTY_MATCH_FAIL_NO_ITEM, vnum)`
removes the player from SearchMap.

Client processing:
- `__PartyMatchMsg(FAIL_NO_ITEM,...)` displays the error and resets the main Party Match state via `__Init()`;
- then `__PartyMatchMinimapButton` runs;
- that function hides the icon only for `PARTY_MATCH_CANCEL_SUCCESS`, `PARTY_MATCH_SUCCESS`, and generic `PARTY_MATCH_FAIL`.

`PARTY_MATCH_FAIL_NO_ITEM` is omitted.

Result: the minimap icon remains visible although matchmaking is no longer active on the server and the main UI state has reset.

This has normal reachability because required items are not reserved/locked for the duration of searching.


## Static closure
Party Match static audit closed on 2026-09-26.

Verified bugs:
- BUG-PMATCH-001 — same-channel search pool fragmented per game core.
- BUG-PMATCH-002 — exchange-listed required item can be destroyed while CExchange retains its raw pointer.
- BUG-PMATCH-003 — queued FAIL_NO_ITEM leaves the minimap Party Match icon stale/visible.

Not promoted:
- duplicate SEARCH/HOLD desync: requires a second SEARCH producer not present in stock UI;
- ignored WarpSet result: configured target coordinates resolve correctly in current deployment;
- client off-state helpers: no active producer found;
- alternate country/ae Party Match config: active loader uses locale/common.
