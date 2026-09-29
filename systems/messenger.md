# Messenger / Friend / Block — Static Mapping

**Status:** MAPPING IN PROGRESS  
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

## Current cursor
Continue static audit of:
- remaining relation symmetry and remove-all paths after BUG-MSG-007;
- local vs P2P presence consistency;
- block enforcement outside whisper (invite/social surfaces);
- client packet-length/state handling;
- GM messenger cache lifecycle;
- logout/reconnect and channel-change boundaries.

Do not execute runtime tests. Global first future live gate remains `DUNGEON-T09`.


## Social-surface pass
- Global shout delivery is filtered per receiver with `IsBlocked(receiver,sender)` in `FuncShout`, including P2P-delivered shouts.
- Direct party invite checks both block directions in `CInputMain::PartyInvite`.
- Guild invite checks both block directions before `CGuild::Invite`.
- Exchange, PvP and equipment-view paths also check messenger blocks.
- The target-board “request to join party” path is different: `uitarget.py::__OnRequestParty -> /party_request -> do_party_request -> CHARACTER::RequestToParty`. The final server path never checks messenger block state. See BUG-MSG-008.
- The previously suspected block-by-VID return-size mismatch is closed: `TPacketCGMessengerAddByVID` and `TPacketCGMessengerAddBlockByVID` are both a single `uint32_t vid`.
- `RemoveAllBlockList` has asymmetric incoming-row cleanup semantics. Active reachability is proven through `questlua_pc.cpp::pc_change_name`; whether the subsequent rename/logout lifecycle fully masks or exposes the stale incoming RAM/P2P state remains under assessment.
- The missing local `m_poMessengerHandler` guard in `OnBlockLogin` is closed as non-bug because `PyCallClassMemberFunc` performs its own null-class guard.

## Updated cursor
Continue static audit of:
- rename/remove-all lifecycle after the now-proven `pc_change_name -> RemoveAllList/RemoveAllBlockList` caller;
- pending party/guild invite acceptance after a new messenger block is established;
- GM messenger cache lifecycle beyond the already verified teardown leak;
- channel-change/reconnect interaction with asynchronous loads and relation cache reconstruction;
- remaining client messenger parser/state boundaries after BUG-MSG-009.

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
