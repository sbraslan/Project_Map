# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only  
**Active state:** Static mapping continuation  
**Status:** MESSENGER / FRIEND / BLOCK — MAPPING IN PROGRESS / 15 VERIFIED BUGS / EXECUTION LOCKED  
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
- `BUG-MSG-010` — pending party/guild invitations are not revalidated against a newly established messenger block at acceptance time.
- `BUG-MSG-011` — `pc.is_blocked` / `pc.is_friend` reject ordinary player-name strings because they require `lua_isnumber` before `FindPC(name)`.
- `BUG-MSG-012` — `RecvMessenger()` still uses a 25-byte legacy name buffer while the server permits 48-character names, allowing stack overwrite on longer messenger names.
- `BUG-MSG-013` — the GM messenger SQL omits the valid `WIZARD` authority even though DB admin loading maps it to `GM_WIZARD`.
- `BUG-MSG-014` — name-based friend/block add paths bypass the observer-mode rejection enforced by the VID paths.
- `BUG-MSG-015` — GM inverse watcher sets retain logged-out accounts and grow with historical process-visible accounts.
- `BUG-MSG-016` — delayed old-core P2P logout can delete the newer same-name channel/session CCI and messenger presence.
- `BUG-MSG-017` — outgoing name-based messenger CG packets truncate the configured 48-byte name boundary and long remove/unblock fields lack explicit final NUL termination.

## Closed / scoped observations
- Shout delivery checks the receiver's messenger block relation on both local and P2P fanout. BUG-MSG-007 can still undermine it after logout/relog cache loss.
- Normal party invite, guild invite, exchange, PvP and equipment-view entry points contain messenger block guards.
- Block-add-by-VID return paths using `sizeof(TPacketCGMessengerAddByVID)` are not a packet-consumption bug in this snapshot because both VID payload structs are one `uint32_t` and therefore equal-sized.
- `RemoveAllBlockList` is reached by `pc.change_name`, but the deployed success path immediately executes `command("quit")`; P2P logout reaches `MessengerManager::Logout` and clears the old-name cache edges. Persistent rename desync is closed as non-promoted/transient.
- `OnBlockLogin`'s missing local handler-null guard is closed as non-bug because `PyCallClassMemberFunc` safely rejects a null handler.

## Exact resume cursor
1. Finish remaining GM messenger cache/login/logout lifecycle after BUG-MSG-013.
2. Audit remaining client/server Messenger packet validation after BUG-MSG-012.
3. Check practical quest usage/reachability of BUG-MSG-011.
4. Run one final relation/cache/P2P symmetry pass and decide whether Messenger is STATIC COMPLETE.
5. Do not execute runtime tests; global first future live gate remains `DUNGEON-T09`.
