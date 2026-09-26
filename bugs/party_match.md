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
