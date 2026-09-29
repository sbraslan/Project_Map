# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Static mapping continuation  
**Status:** MESSENGER / FRIEND / BLOCK — MAPPING IN PROGRESS / 19 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Messenger / Friend / Block  
**System:** `systems/messenger.md`  
**Bugs:** `bugs/messenger.md`  
**Tests:** `tests/messenger.md`  
**Previous completed subsystem:** Mining / Pickaxe  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Verified findings
- `BUG-MSG-001..019` are canonical in `bugs/messenger.md`.
- Latest additions: `BUG-MSG-018` Battle Field add-by-VID parity bypass; `BUG-MSG-019` pending friend acceptance lacks messenger-block revalidation.
- Runtime plans are `MSG-T01..MSG-T19`; none has been executed.

## Closed / scoped observations
- Shout delivery checks the receiver's messenger block relation on both local and P2P fanout. BUG-MSG-007 can still undermine it after logout/relog cache loss.
- Normal party invite, guild invite, exchange, PvP and equipment-view entry points contain messenger block guards.
- Block-add-by-VID return paths using `sizeof(TPacketCGMessengerAddByVID)` are not a packet-consumption bug in this snapshot because both VID payload structs are one `uint32_t` and therefore equal-sized.
- `RemoveAllBlockList` is reached by `pc.change_name`, but the deployed success path immediately executes `command("quit")`; P2P logout reaches `MessengerManager::Logout` and clears the old-name cache edges. Persistent rename desync is closed as non-promoted/transient.
- `OnBlockLogin`'s missing local handler-null guard is closed as non-bug because `PyCallClassMemberFunc` safely rejects a null handler.

## Exact resume cursor
1. Audit friend/block remove and inverse-cache symmetry after BUG-MSG-019.
2. Finish remaining GM messenger lifecycle and parser boundaries.
3. Keep malformed fixed-width name parsing candidate-only unless ordinary reachability is proven.
4. Perform final Messenger relation/cache/P2P symmetry pass and decide STATIC COMPLETE.
5. Do not execute runtime tests; global first future live gate remains `DUNGEON-T09`.
