# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Static mapping continuation  
**Status:** MESSENGER / FRIEND / BLOCK — MAPPING IN PROGRESS / 9 VERIFIED BUGS / EXECUTION LOCKED  
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
- `BUG-MSG-001` — block-add friend validation duplicates `IsBlocked`, allowing friend + block coexistence.
- `BUG-MSG-002` — remote/P2P whispers bypass messenger block enforcement because checks require a local `pkChr`.
- `BUG-MSG-003` — delayed async messenger DB callbacks can repopulate state after logout.
- `BUG-MSG-004` — client `OnBlockLogin` calls the logout UI callback.
- `BUG-MSG-005` — client teardown leaves block/GM messenger caches alive across sessions.
- `BUG-MSG-006` — pending friend authorization has no server-side expiry/logout cleanup.
- `BUG-MSG-007` — companion logout erases persistent outgoing friend/block cache for still-online users; relog does not reconstruct it.
- `BUG-MSG-008` — client-visible `/party_request` route bypasses messenger block checks that protect the normal party-invite packet route.
- `BUG-MSG-009` — unblock-by-VID can dereference a vanished target instance after the confirmation delay.

## Closed / scoped observations
- Shout delivery checks the receiver's messenger block relation on both local and P2P fanout. BUG-MSG-007 can still undermine it after logout/relog cache loss.
- Normal party invite, guild invite, exchange, PvP and equipment-view entry points contain messenger block guards.
- Block-add-by-VID return paths using `sizeof(TPacketCGMessengerAddByVID)` are not a packet-consumption bug in this snapshot because both VID payload structs are one `uint32_t` and therefore equal-sized.
- `RemoveAllBlockList` has a DB-vs-cache/P2P asymmetry for incoming block rows, but no active call site has yet been established; keep it candidate-only.

## Exact resume cursor
1. Continue client messenger parser/state safety: list-length accounting and optimistic unblock/server-ack behavior.
2. Finish GM messenger cache/login/logout lifecycle symmetry.
3. Re-check channel-change boundaries against BUG-MSG-003/007 and P2P presence reconstruction.
4. Resolve the dormant `RemoveAllBlockList` call-site question before any promotion.
5. Do not execute runtime tests; global first future live gate remains `DUNGEON-T09`.
