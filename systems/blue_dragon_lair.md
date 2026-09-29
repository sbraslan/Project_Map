# Blue Dragon / Beran Setaou — Static System Map

**Status:** STATIC MAPPING OPEN / 0 VERIFIED BUGS
**Mode:** detection / mapping only
**Execution:** LOCKED / NOT RUN

## Scope
Active Blue Dragon / Beran Setaou renewal flow: quest entry/state, map 208 ownership, global event state, access-item transaction, BlueDragon combat hooks, skill factors, lair regen/data and compiled quest handlers.

Meley / Red Dragon Lair is a separate canonical family because it has its own manager, participant, reward and ranking lifecycle.

The older `CDragonLairManager` / `DragonLair.startRaid` surface is dependency-only until a tracked deployed caller is proven.

## Audit cursor
1. Entry authority, global event state, channel serialization, group-entry window and access-item transaction.
2. Boss spawn/combat hooks, skill timers and stone modifiers.
3. Timeout, death, rejoin/login and room purge/warp lifecycle.
4. Legacy `DragonLair.startRaid` reachability.
5. Deployment/data parity and static closure.
