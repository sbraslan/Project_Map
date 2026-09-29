# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Static mapping continuation  
**Status:** MESSENGER / FRIEND / BLOCK ACTIVE — SERVER ENTRYPOINT PASS 1 COMPLETE / 1 VERIFIED BUG / EXECUTION LOCKED  
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

## Current verified finding
- `BUG-MSG-001` — both block-add-by-VID and block-add-by-name duplicate `IsBlocked(actor,target)` where the first guard/message is the friend-list guard. Existing friends can therefore pass both checks and be added to the block relation without removing the friend relation.

## Exact resume cursor
1. Trace P2P propagation and login/logout reconstruction for simultaneous friend+block state.
2. Trace client packet/UI entry points for both block-add variants.
3. Check whisper/shout/party/guild behavior when friend+block coexist.
4. Audit block-add-by-VID return-size mismatches before deciding whether they are a real defect.
5. Do not execute runtime tests; global first future live gate remains `DUNGEON-T09`.
