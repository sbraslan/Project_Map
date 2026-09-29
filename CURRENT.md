# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Static mapping continuation  
**Status:** MESSENGER / FRIEND / BLOCK ACTIVE — P2P + LOGIN/LOGOUT PASS COMPLETE / 2 VERIFIED BUGS / EXECUTION LOCKED  
**Machine state:** `STATE.json`  
**Active subsystem:** Messenger / Friend / Block  
**System:** `systems/messenger_friend_block.md`  
**Bugs:** `bugs/messenger_friend_block.md`  
**Tests:** `tests/messenger_friend_block.md`  
**Previous completed subsystem:** Mining / Pickaxe  
**First future live gate:** `DUNGEON-T09`  
**Last updated:** 2026-09-29

## Hard rule
Only `sbraslan/Project_Map` is writable. Source/game repositories remain read-only. Runtime/fault-injection execution remains locked until an explicit phase change.

## Verified findings
- `BUG-MSG-001` — both block-add-by-VID and block-add-by-name duplicate `IsBlocked(actor,target)` where the first guard/message is intended to reject an existing friend relation. Simultaneous friend+block state can be persisted and propagated.
- `BUG-MSG-002` — when a blocked target logs out, `Logout(target)` erases that target from all other users' in-memory block sets; target relog loads only its own outgoing block rows, so a still-online blocker's `IsBlocked(blocker,target)` can remain false despite the persistent DB row.

## Exact resume cursor
1. Trace client packet/UI entry points for block add-by-VID, block add-by-name, and remove.
2. Check whisper/shout/party/guild behavior after BUG-MSG-002 desync.
3. Audit block-add-by-VID packet return-size mismatches before promotion.
4. Do not execute runtime tests; global first future live gate remains `DUNGEON-T09`.
