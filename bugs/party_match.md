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
