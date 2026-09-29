# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Messenger static closure / global coverage discovery next  
**Status:** MESSENGER / FRIEND / BLOCK — STATIC COMPLETE / 21 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Messenger / Friend / Block — CLOSED  
**System:** `systems/messenger.md`  
**Bugs:** `bugs/messenger.md`  
**Tests:** `tests/messenger.md`  
**Previous completed subsystem:** Messenger / Friend / Block  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Verified findings
- `BUG-MSG-001..021` are canonical in `bugs/messenger.md`.
- Latest additions: `BUG-MSG-020` 16-bit list-size framing overflow; `BUG-MSG-021` GM-to-GM block route-policy mismatch.
- Runtime plans are `MSG-T01..MSG-T21`; none has been executed.

- `BUG-MSG-018` — Battle Field rejects friend add-by-name but the target-board add-by-VID path lacks the same map restriction.

- `BUG-MSG-019` — pending friend authorization can still be accepted after either side establishes a messenger block.

- `BUG-MSG-020` — oversized friend/block lists can overflow the 16-bit messenger packet-size field while the full payload is still transmitted.

- `BUG-MSG-021` — GM-to-GM block authorization differs between VID and name routes.

## Closed / scoped observations
- Shout delivery checks the receiver's messenger block relation on both local and P2P fanout. BUG-MSG-007 can still undermine it after logout/relog cache loss.
- Normal party invite, guild invite, exchange, PvP and equipment-view entry points contain messenger block guards.
- Block-add-by-VID return paths using `sizeof(TPacketCGMessengerAddByVID)` are not a packet-consumption bug in this snapshot because both VID payload structs are one `uint32_t` and therefore equal-sized.
- `RemoveAllBlockList` is reached by `pc.change_name`, but the deployed success path immediately executes `command("quit")`; P2P logout reaches `MessengerManager::Logout` and clears the old-name cache edges. Persistent rename desync is closed as non-promoted/transient.
- `OnBlockLogin`'s missing local handler-null guard is closed as non-bug because `PyCallClassMemberFunc` safely rejects a null handler.

## Static closure
- Messenger / Friend / Block is closed statically with `BUG-MSG-001..021`.
- Deferred runtime plans are `MSG-T01..MSG-T21`; none has been executed.
- Final remove/inverse/P2P symmetry review produced no additional independent finding.
- CRC collision and marriage/block policy observations remain unpromoted.
- Global first future live gate remains `DUNGEON-T09`.

## Exact resume cursor
1. Open the next independent subsystem for detection-only mapping.
2. Keep all source/game repositories read-only.
3. Do not execute runtime or fault-injection tests.
