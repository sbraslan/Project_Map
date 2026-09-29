# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Marriage / Wedding static mapping  
**Status:** MARRIAGE / WEDDING — STATIC MAPPING IN PROGRESS / 6 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Marriage / Wedding — OPEN  
**System:** `systems/marriage.md`  
**Bugs:** `bugs/marriage.md`  
**Tests:** `tests/marriage.md`  
**Previous completed subsystem:** Messenger / Friend / Block  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Verified findings
- `BUG-MARR-001` — engagement Yang/ring mutations are committed before the authoritative marriage request and an intervening `wait()` creates an interruption window with no rollback.
- `BUG-MARR-002` — map 81 is enabled on both ch2/core4 and ch99/core99 while DB wedding requests are broadcast to all game peers, allowing duplicate wedding-map producers/READY paths.
- `BUG-MARR-003` — mutual divorce incorrectly rejects an exactly sufficient 500,000 Yang balance.
- `BUG-MARR-004` — delayed old-core P2P logout can overwrite a fresh lover-online state.
- `BUG-MARR-005` — wedding exit ignores the actual saved exit fields and uses warp fields instead.
- `BUG-MARR-006` — same-core wedding exit does not detach membership, allowing delayed teardown to disconnect an already-exited player.
- Deferred Marriage tests: `MARR-T01..MARR-T06`; none executed.

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
1. Finish mutual-divorce stale-target/reselection audit.
2. Close near-check/love-point lifecycle.
3. Close wedding end + membership interactions after BUG-MARR-005/006.
4. Audit remaining Lua state assumptions and marriage unique-item bonus semantics.
5. Decide STATIC COMPLETE readiness; keep runtime locked.

## Mapping acceleration index — 2026-09-29
- Status: **READY**
- Canonical directory: `index/`
- Source repos remain **READ-ONLY**.
- GitHub Code Search is no longer a hard dependency; recursive Git tree inventory + Project_Map machine index is the primary lookup path.
- Indexed mapping-relevant files: **10,119**.
- Seed coverage: **34 systems / 228 symbols / 234 call-flow edges**.
- Context7 role: external dependency/API verification only.
- Next mapping action: use the new index for global source-feature coverage discovery and choose the next unmapped/under-mapped subsystem.
