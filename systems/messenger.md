# Messenger / Friend / Block — Static Mapping

**Status:** STATIC COMPLETE  
**Mode:** Detection / mapping only  
**Runtime execution:** LOCKED  
**Source repositories:** READ-ONLY  
**Writable repository:** `sbraslan/Project_Map`

## Scope mapped so far

### Server
- `common/CommonDefines.h` — `ENABLE_MESSENGER_BLOCK` and `ENABLE_GM_MESSENGER_LIST` are enabled.
- `game/src/messenger_manager.cpp/.h` — friend/block relation caches, DB load, request/auth lifecycle, P2P replication.
- `game/src/input_main.cpp` — messenger CG dispatcher, friend add/remove, block add/remove, whisper block enforcement.
- `game/src/input_login.cpp` — messenger login/load entry point.
- `game/src/char.cpp` — messenger logout lifecycle.
- `game/src/p2p.cpp` and `input_p2p.cpp` — cross-core login/logout and relation replication.
- `game/src/cmd.cpp` + `cmd_general.cpp` — `/messenger_auth` registration and friend-request response.
- `game/src/db.cpp` — `FuncQuery` is asynchronous through ReturnQuery/Process.
- `game/src/packet.h` — messenger friend/block packet families.

### Client / Binary
- `Project_ClientSrc/UserInterface/PythonMessenger.cpp/.h` — local friend/guild/GM/block caches and Python callbacks.
- `Project_ClientSrc/UserInterface/PythonNetworkStreamPhaseGame.cpp` — GC messenger receive path plus block add/remove send paths.
- `Project_ClientSrc/UserInterface/PythonCharacterManager.cpp` — VID lookup semantics (`GetInstancePtr` returns null for absent instances).
- `Project_Binary/root/uimessenger.py` — friend/guild/GM/block UI groups and online/offline rendering.
- `Project_Binary/root/game.py` — friend authorization dialog and `messenger.Destroy()` teardown.
- `Project_Binary/root/uitarget.py` — block/unblock target-board confirmation and social target actions.

## Core flow

### Login
`CInputLogin::Entergame` -> `MessengerManager::Login(name)` -> async SQL friend/GM/block list queries -> callbacks populate process-local relation caches -> GC messenger lists update client.

### Friend add
Client add-by-VID/name -> `CInputMain::Messenger` -> `RequestToAdd` -> CRC pair inserted into `m_set_requestToAdd` -> target receives `messenger_auth` command -> UI accept/deny -> `/messenger_auth y|n <name>` -> `AuthToAdd` -> symmetric `AddToList` on accept -> DB + P2P replication.

### Block add
Client block-by-VID/name -> `CInputMain::Messenger` -> validation -> `AddToBlockList` -> DB + local cache + P2P replication.

### Whisper enforcement
Local target: `CInputMain::Whisper` can query messenger block relation in the current process.  
Remote target: target is represented by P2P `CCI` and relay descriptor while `pkChr == nullptr`.

## Verified findings
- `BUG-MSG-001` — block-add friend check accidentally duplicates `IsBlocked`, allowing friend+block coexistence and shadowing the intended already-blocked branch.
- `BUG-MSG-002` — messenger block enforcement is skipped for P2P/remote whispers because both checks require a local `pkChr`.
- `BUG-MSG-003` — asynchronous list callbacks can repopulate relation state and emit presence after logout.
- `BUG-MSG-004` — client `OnBlockLogin` dispatches `OnLogout`, so online blocked users render offline.
- `BUG-MSG-005` — client `Destroy()` leaves block and GM caches intact across game-window/session teardown.
- `BUG-MSG-006` — pending friend authorization tokens have no server timeout/logout cleanup and can be consumed later.
- `BUG-MSG-007` — logout erases the departing character from every online account's outgoing friend/block cache; reconnect reloads only the departing account, so persistent block/friend state is not restored for observers.
- `BUG-MSG-008` — target-board `/party_request` uses a player command path without messenger block validation, bypassing the guarded direct party-invite path.
- `BUG-MSG-009` — target-board unblock confirmation can outlive the target instance; remove-by-VID dereferences `GetInstancePtr(vid)` without a null guard.
- `BUG-MSG-012` — project-wide character names were extended to 48, but `RecvMessenger()` still receives every messenger name into a 25-byte legacy stack buffer.
- `BUG-MSG-010` — pending party/guild invitations can still be accepted after a messenger block is established because acceptance does not revalidate block state.
- `BUG-MSG-011` — `pc.is_blocked` and `pc.is_friend` are registered Lua name-query helpers but gate their first argument with `lua_isnumber` before calling `FindPC(name)`.
- `BUG-MSG-013` — the server recognizes `WIZARD` as `GM_WIZARD`, but the Messenger GM-list SQL omits `mAuthority='WIZARD'`.
- `BUG-MSG-014` — add-by-name friend/block paths omit the observer-mode rejection enforced by their VID counterparts, and both name paths are exposed by the Messenger UI.
- `BUG-MSG-015` — synthetic GM inverse watcher sets retain historical logged-out accounts because logout does not prune `m_InverseGMRelation[gm]`.
- `BUG-MSG-016` — delayed old-core P2P logout can remove a newer same-name CCI/session presence after channel/core handoff.
- `BUG-MSG-017` — client→server name-based messenger packets use a 48-byte field for a configured 48-byte name, truncating add/block-add to 47 bytes and leaving long remove/unblock fields without an explicit final NUL.
- `BUG-MSG-018` — Battle Field blocks friend add-by-name but the normal target-board add-by-VID path lacks the map restriction.
- `BUG-MSG-019` — pending friend authorization revalidates only its request token, not messenger block state, so a block added during the pending window does not prevent acceptance.
- `BUG-MSG-020` — friend/block list serialization can exceed the 16-bit `TPacketGCMessenger::size` while the server still transmits the full buffer, desynchronizing client packet framing.
- `BUG-MSG-021` — GM-to-GM block authorization differs between block-by-VID and block-by-name routes.

## Static closure
Messenger / Friend / Block is **STATIC COMPLETE** with `BUG-MSG-001..021` and deferred tests `MSG-T01..MSG-T21`.

Coverage closed across:
- friend add/remove/request authorization and pending-token lifecycle;
- block add/remove, enforcement and UI state;
- local/P2P whisper and social interaction surfaces;
- login/logout, async DB callbacks and relation-cache reconstruction;
- cross-core/channel presence ordering;
- GM messenger query/cache lifecycle;
- client send/receive packet boundaries and long-name compatibility.

Two theoretical robustness cases remain deliberately unpromoted: CRC-pair token collision and fully crafted non-terminated fixed-width CG fields beyond the ordinary client contract.

Do not execute runtime tests. Global first future live gate remains `DUNGEON-T09`.


## Social-surface pass
- Global shout delivery is filtered per receiver with `IsBlocked(receiver,sender)` in `FuncShout`, including P2P-delivered shouts.
- Direct party invite checks both block directions in `CInputMain::PartyInvite`.
- Guild invite checks both block directions before `CGuild::Invite`.
- Exchange, PvP and equipment-view paths also check messenger blocks.
- The target-board “request to join party” path is different: `uitarget.py::__OnRequestParty -> /party_request -> do_party_request -> CHARACTER::RequestToParty`. The final server path never checks messenger block state. See BUG-MSG-008.
- The previously suspected block-by-VID return-size mismatch is closed: `TPacketCGMessengerAddByVID` and `TPacketCGMessengerAddBlockByVID` are both a single `uint32_t vid`.
- `RemoveAllBlockList` is actively reached through `questlua_pc.cpp::pc_change_name`, but the deployed name-change quest immediately issues `command("quit")` after success. P2P logout calls `MessengerManager::P2PLogout -> Logout`, which removes the old name from all outgoing relation/block caches. Persistent rename desync is therefore closed; only a short transient window exists.
- The missing local `m_poMessengerHandler` guard in `OnBlockLogin` is closed as non-bug because `PyCallClassMemberFunc` performs its own null-class guard.

## Updated cursor
Continue static audit of:
- pending friend-token identity/collision characteristics (candidate only unless ordinary reachability is established);
- malformed/fixed-width messenger packet parsing (candidate only unless ordinary server-generated reachability is established);
- residual relation/cache/P2P symmetry after BUG-MSG-015/016;
- final static-closure readiness.

Do not execute runtime tests. Global first future live gate remains `DUNGEON-T09`.


### Client block-remove lifetime
The target-board unblock confirmation stores only the target VID across the confirmation delay. The eventual C++ remove-by-VID path resolves that VID again but does not validate the returned instance pointer before reading its name. This creates BUG-MSG-009 when the target disappears before confirmation.

The same send path removes the local block entry immediately and has no positive server acknowledgement. If server-side `IsBlocked` has already false-negatived because of BUG-MSG-007, the server returns without deleting the persistent row while the client has already removed it from its local set. This is tracked as an explicit BUG-MSG-007 consequence for runtime verification.


## Client unblock stale-VID pass
- `uitarget.py::__OnBlockRemove` opens a confirmation dialog without snapshotting/validating the target identity at accept time.
- `TargetBoard.Close -> __Initialize` resets `self.vid` to zero but does not close that confirmation.
- `SendMessengerBlockRemoveByVIDPacket` resolves the VID through `CPythonCharacterManager::GetInstancePtr` and dereferences the result with no null check.
- `GetInstancePtr` returns `nullptr` for a missing VID.
- Result: verified `BUG-MSG-009`; deferred test `MSG-T09`.


### Messenger name-length compatibility
Both server and client canonical character-name constants are 48 in this snapshot, while `RecvMessenger()` retains the pre-extension literal `char_name[24 + 1]`. All friend/GM/block/mobile variable-length name branches trust the packet length and write a terminator at that index. This is BUG-MSG-012.

### Channel-change ordering pass
`MoveChannel -> WarpSet(custom addr/port)` sends `GC_WARP`; the destination process later broadcasts a new `GG_LOGIN`, while the source process broadcasts `GG_LOGOUT` as the old character disconnects. On an observer process, `P2P_MANAGER::Login` updates an existing same-name CCI in place, but `CInputP2P::Logout` removes by name only and does not verify that the logout came from the descriptor/session currently stored in that CCI. Because old logout and new login originate from different P2P peer connections, no global packet order is guaranteed. LOGIN(new) followed by delayed LOGOUT(old) therefore deletes the newer CCI and runs messenger logout against the still-online identity. This is now `BUG-MSG-016`; deferred validation is `MSG-T16`.


### GM inverse lifecycle pass
Every account login synthesizes outgoing GM relations and also inserts the account into each GM's `m_InverseGMRelation` watcher set. Logout erases the account's outgoing GM relation but never removes that account from the GM inverse sets, and `MessengerManager::Destroy()` is empty. This produces `BUG-MSG-015`; deferred validation is `MSG-T15`.

### Observer-mode name-path parity
The friend and block VID branches explicitly reject observer mode. Their name-based counterparts do not, while `uimessenger.py` exposes both friend-by-name and block-by-name inputs without an observer gate. This is `BUG-MSG-014`; deferred validation is `MSG-T14`.


### Client-to-server long-name packet boundary
The outgoing messenger helpers do not reserve `CHARACTER_NAME_MAX_LEN + 1` like the canonical player-name representation. Add-by-name and block-add-by-name deliberately terminate byte 47 and therefore cannot encode a full 48-byte configured name. Remove/unblock helpers copy at most 47 bytes into a 48-byte stack array but omit an explicit last-byte terminator; at the long-name boundary the transmitted field may contain a stale final byte. This is tracked as `BUG-MSG-017`, separate from the server→client overflow in `BUG-MSG-012`.

### Deployment/reachability qualification
A repository-wide Project_Game search found no tracked quest caller for `pc.is_blocked` or `pc.is_friend`; `BUG-MSG-011` remains a verified registered Lua API defect, but no currently tracked deployed quest depends on it.

### Observer-mode path parity
Friend/block add-by-VID rejects observer-mode characters. The corresponding name-entry actions from `uimessenger.py` reach add-by-name server branches with no observer-mode guard, while `HEADER_CG_MESSENGER` is not globally observer-gated. This is verified `BUG-MSG-014`.

### Pending friend authorization revalidation
- Friend-request creation checks messenger block in both directions before `RequestToAdd`.
- `RequestToAdd` stores a pending CRC token and surfaces `messenger_auth` to the recipient.
- `/messenger_auth y` reaches `AuthToAdd`, which checks only the token and then creates both friend edges.
- No `IsBlocked` revalidation occurs at acceptance time.
- Result: verified `BUG-MSG-019`; deferred test `MSG-T19`.


### Friend-authorization TOCTOU
The initial add-by-VID/name branches enforce block state, but the pending request token survives later block creation. `AuthToAdd` checks only token existence and then writes both friend directions. This is `BUG-MSG-019`, independent of the block-add validation typo in `BUG-MSG-001`.

### GM inverse-cache lifecycle
`LoadGMList` creates synthetic account→GM relations and inverse GM→account memberships. Logout removes the forward list but not the inverse memberships; `Destroy()` is empty. Therefore the inverse side grows with distinct historical logins and GM presence fanout traverses stale accounts. Verified as `BUG-MSG-015`.


### Oversized messenger-list framing
`TPacketGCMessenger::size` is 16-bit, but `SendList` and `SendBlockList` construct a complete relation payload in a 128 KiB temporary buffer with no cumulative size or relation-count cap. `pack.size += buf.size()` can therefore wrap while the full `buf.size()` bytes are still transmitted. Client parsing uses the wrapped advertised size, leaving surplus bytes to corrupt later stream framing. This is `BUG-MSG-020`; deferred validation is `MSG-T20`.

### GM block route parity
The block-by-VID branch rejects a GM target only when the requester is `GM_PLAYER`, while block-by-name rejects every GM target on non-test servers regardless of requester authority. Target-board and Messenger name-entry surfaces expose both routes and no common later guard reconciles them. This is `BUG-MSG-021`; deferred validation is `MSG-T21`.


### Oversized list framing
`TPacketGCMessenger::size` is 16-bit while friend/block list builders use a 128 KiB buffer and enforce no cumulative relation-count/packet-size ceiling. Once a list exceeds 65,535 bytes the advertised size wraps but the full buffer is still sent. This is `BUG-MSG-020`.

### Mobile/SMS deployment scope
- `PythonNetworkStreamModule.cpp::netSendMobileMessagePacket` is a no-op stub.
- `uimessenger.py` contains the legacy mobile-message UI, but `uigameoption.py` defaults `MOBILE = False` and enables it only for `localeInfo.IsYMIR()`.
- Current Europe deployment therefore has no ordinary UI reachability for the broken send binding; this is closed as dormant/Ymir-only and is not assigned a bug ID.

## Static closure
Messenger / Friend / Block is **STATIC COMPLETE** with canonical `BUG-MSG-001..021` and deferred `MSG-T01..MSG-T21`.
No runtime/fault-injection test has been executed. Global first future live gate remains `DUNGEON-T09`.
