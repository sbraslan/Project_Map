# Messenger / Friend / Block — Verified Bugs

**Status:** MAPPING IN PROGRESS / VERIFIED STATIC FINDINGS  
**Execution:** LOCKED

## BUG-MSG-001 — block-add friend validation duplicates the block predicate

**Class:** state consistency / validation defect  
**Reachability:** VERIFIED through both block-by-VID and block-by-name packet paths.

### Proof
In both `MESSENGER_SUBHEADER_CG_BLOCK_ADD_BY_VID` and `MESSENGER_SUBHEADER_CG_BLOCK_ADD_BY_NAME`:
1. the first predicate calls `IsBlocked(account, target)` but emits “Remove ... from your friends list to continue.”;
2. the immediately following predicate calls the exact same `IsBlocked(account, target)` and emits “already being blocked”;
3. no `IsInList(account, target)` check is performed before `AddToBlockList`.

### Consequence
A player already present in the friend list can also be inserted into the block list. The second “already blocked” branch is also unreachable because the first identical condition returns first.

### Deferred validation
`MSG-T01`.

---

## BUG-MSG-002 — remote/P2P whispers bypass messenger block enforcement

**Class:** multi-core authorization / communication bypass  
**Reachability:** VERIFIED for a whisper target resolved through P2P rather than local `FindPC`.

### Proof
- Local targets populate `pkChr`.
- Remote targets populate a P2P `CCI` / relay descriptor while `pkChr` remains null.
- Both messenger block checks in `CInputMain::Whisper` are guarded by `pkChr && ...`.
- The remote relay path therefore reaches normal whisper forwarding without consulting the replicated block relation.

### Consequence
A messenger block can stop whispers while both characters are on the same process yet fail when they are separated across cores/channels.

### Deferred validation
`MSG-T02`.

---

## BUG-MSG-003 — async messenger list callbacks can resurrect state after logout

**Class:** async lifecycle / stale presence state  
**Reachability:** VERIFIED as a static race when logout happens before a queued SQL callback is processed.

### Proof
1. `MessengerManager::Login` schedules friend/GM/block loads with `DBManager::FuncQuery`.
2. `FuncQuery` queues a ReturnQuery and callbacks execute later in `DBManager::Process`.
3. `Login` marks the account logged in before those callbacks return.
4. `Logout` removes login state and clears relation caches.
5. `LoadList`, `LoadGMList`, and `LoadBlockList` do not re-check whether the account is still logged in.
6. A late callback can repopulate relation/inverse maps; `LoadList` can additionally send login notifications through inverse relations.

### Consequence
Fast disconnect/reconnect or delayed DB response can leave stale relation data and false presence notifications after logout.

### Deferred validation
`MSG-T03`.

---

## BUG-MSG-004 — client block-login callback marks online blocked users offline

**Class:** client state/UI callback defect  
**Reachability:** VERIFIED for block-list entries reported online.

### Proof
`CPythonMessenger::OnBlockLogin` inserts the name into `m_BlockNameMap` and invokes the Python handler method `OnLogout`, not `OnLogin`.

`uimessenger.py::OnLogout` explicitly marks the member offline.

### Consequence
Blocked players who are online are rendered/maintained as offline in the messenger block group.

### Deferred validation
`MSG-T04`.

---

## BUG-MSG-005 — client teardown does not clear block or GM messenger caches

**Class:** cross-session client state leak  
**Reachability:** VERIFIED across game-window teardown without terminating the client process.

### Proof
- `CPythonMessenger` owns friend, guild, optional GM, and optional block collections.
- `CPythonMessenger::Destroy()` clears only `m_FriendNameMap` and `m_GuildMemberStateMap`.
- It does not clear `m_BlockNameMap` or `m_GMNameMap`.
- `root/game.py::Close` calls `messenger.Destroy()`.
- A subsequent server login with no rows can return from the load callback without sending a clearing list, so stale names need not be overwritten.

### Consequence
Block/GM entries from a previous character/account session can remain in the client singleton and influence later UI/client queries until process exit or explicit removal.

### Deferred validation
`MSG-T05`.

---

## BUG-MSG-006 — pending friend requests have no server-side expiry or logout cleanup

**Class:** stale authorization token / lifecycle defect  
**Reachability:** VERIFIED after a friend request is created and the recipient disconnects or otherwise does not answer.

### Proof
1. `RequestToAdd` inserts a CRC-derived pair into `m_set_requestToAdd`.
2. The only observed erase occurs inside `AuthToAdd`.
3. `Logout` does not clear pending request entries and no timeout is attached.
4. `messenger_auth` is registered as a normal `GM_PLAYER` command.
5. A later `/messenger_auth y <requester>` with the same pair can still consume the stale token.
6. `AuthToAdd` does not require the original requester to still be online before writing both friend relations.

### Consequence
A request can remain authorizable long after its original UI/request lifetime, including after disconnect/relogin.

### Deferred validation
`MSG-T06`.

## Earlier candidate notes
- add-by-name observer-mode parity is now promoted as `BUG-MSG-016`.
- remove-all / rename persistence was closed as transient because the deployed successful rename path immediately disconnects and P2P logout clears the old-name cache edges.
- `OnBlockLogin`'s missing local handler-null guard is closed as non-bug because `PyCallClassMemberFunc` itself rejects a null class/handler safely.


---

## BUG-MSG-007 — companion logout destroys persistent outgoing friend/block cache for online users

**Class:** lifecycle / relation-cache corruption  
**Reachability:** VERIFIED when A remains online while B logs out and later reconnects.

### Proof
1. `Logout(B)` iterates every `m_Relation` entry and executes `erase(B)`, then erases `m_Relation[B]`.
2. With messenger block enabled it performs the same global erase of B from every `m_BlockRelation` entry, then erases `m_BlockRelation[B]`.
3. The inverse maps are retained to send B's offline/online presence to observers.
4. When B reconnects, `LoadList(B)` and `LoadBlockList(B)` load only rows whose **account is B**.
5. They do not reload A's outgoing relation/block rows, so `m_Relation[A][B]` and `m_BlockRelation[A][B]` stay missing while A remains online.
6. `IsBlocked(A,B)` and `IsInList(A,B)` read those outgoing maps.

### Consequence
Persistent relationships in SQL diverge from the live server cache after the companion logs out. Most critically, if A blocked B, B logging out and reconnecting can make `IsBlocked(A,B)` false until A itself reloads its list. This creates a same-core block-enforcement bypass in addition to the separate cross-core whisper bypass in BUG-MSG-002. Friend membership checks/duplicate prevention can also observe a false-negative cache. The same cache loss can undermine receiver-side shout filtering. It can also make a later unblock request fail the server-side `IsBlocked` precheck while the client optimistically removes the block locally, leaving DB/client state divergent until reload.

### Deferred validation
`MSG-T07`.


---

## BUG-MSG-008 — party-request command bypasses messenger block enforcement

**Class:** social-interaction authorization bypass  
**Reachability:** VERIFIED through the normal target-board UI.

### Proof
1. The normal `HEADER_CG_PARTY_INVITE` path in `CInputMain::PartyInvite` checks both directions with `MessengerManager::IsBlocked` before sending the party invitation.
2. A separate player command `party_request` is registered at `GM_PLAYER` level.
3. `do_party_request` resolves the target VID and calls `CHARACTER::RequestToParty(tch)`.
4. `CHARACTER::RequestToParty` checks `BLOCK_PARTY_REQUEST`, party state and join conditions, but does not consult messenger `IsBlocked` in either direction.
5. `Project_Binary/root/uitarget.py` exposes this route through `TARGET_BUTTON_REQUEST_ENTER_PARTY`; `__OnRequestParty` sends `/party_request <vid>`.

### Consequence
When A targets B while B is already in a party, A can use the normal target-board “request to join party” surface to deliver a party request even when A and B are messenger-blocked. This is inconsistent with the protected direct party-invite route and bypasses the block relationship on a standard player UI path.

### Deferred validation
`MSG-T08`.

## Candidate / closure notes after social-surface pass
- Shout fanout checks `IsBlocked(receiver,sender)` on every process, including P2P shout delivery. Logout cache corruption from BUG-MSG-007 can still make this check false later; this is an impact extension of BUG-MSG-007, not a separate finding.
- Guild invite, direct party invite, exchange, PvP and equipment-view paths observed in this pass contain messenger block checks.
- The block-add-by-VID branches that sometimes return `sizeof(TPacketCGMessengerAddByVID)` are equal-sized to `TPacketCGMessengerAddBlockByVID` in this snapshot (both contain one `uint32_t vid`), so the suspected packet-consumption mismatch is closed as non-bug.
- `RemoveAllBlockList(account)` is actively reached by `pc.change_name` and is internally asymmetric, but the deployed successful rename quest immediately executes `command("quit")`; P2P logout routes to `MessengerManager::Logout`, which removes the old name from every outgoing relation/block cache. The suspected persistent rename desync is therefore closed as a transient/non-promoted observation.


---

## BUG-MSG-009 — target-board block removal can dereference a missing client instance

**Class:** client crash / stale VID lifetime defect  
**Reachability:** VERIFIED through the normal target-board unblock confirmation flow.

### Proof
1. `uitarget.py::__OnBlockRemove` opens a confirmation dialog whose accept callback later calls `OnBlockRemove` and uses the board's current `self.vid`.
2. `TargetBoard.Close()` calls `__Initialize()`, which resets `self.vid = 0`, but it does not close or invalidate the block-removal confirmation dialog.
3. If the target disappears while the dialog is open, the target board can close/reset while the confirmation remains actionable. Even without the reset path, the stored VID can simply become stale after the instance is removed.
4. Accepting the dialog calls `SendMessengerBlockRemoveByVIDPacket(self.vid)`.
5. That C++ function calls `CPythonCharacterManager::GetInstancePtr(vid)` and immediately dereferences the returned pointer through `pInstance->GetNameString()` without a null check.
6. `GetInstancePtr` explicitly returns `nullptr` when the VID is absent from `m_kAliveInstMap`.

### Consequence
A normal UI race — open “remove block”, let the targeted character leave/despawn or otherwise lose its client instance, then accept — can reach a null-pointer dereference and terminate the client.

### Deferred validation
`MSG-T09`.
---

## BUG-MSG-010 — pending party/guild invites do not revalidate a newly established messenger block

**Class:** TOCTOU / authorization revalidation defect  
**Reachability:** VERIFIED for the existing pending-invite acceptance handlers.

### Proof
1. `CInputMain::PartyInvite` checks `IsBlocked` in both directions before creating the party invite.
2. The answer path is `PartyInviteAnswer -> CHARACTER::PartyInviteAccept`.
3. `PartyInviteAccept` validates the pending event and mutable party conditions but never re-checks messenger block.
4. Guild invite creation likewise checks both messenger-block directions before `CGuild::Invite`.
5. `CGuild::InviteAccept` validates the pending invite event and guild join conditions but does not retain/re-check the original inviter's messenger-block relation.

### Consequence
An invite that was valid when created can still be accepted after either side establishes a messenger block during the pending window.

### Deferred validation
`MSG-T10`.

---

## BUG-MSG-011 — Lua friend/block query helpers reject ordinary player-name strings

**Class:** Lua API / argument validation defect  
**Reachability:** VERIFIED for the registered `pc.is_blocked` and `pc.is_friend` functions.

### Proof
1. Both helpers are registered under `ENABLE_MESSENGER_BLOCK`.
2. Both start with `lua_isnumber(L, 1)` and return without a boolean if the check fails.
3. Immediately afterward they call `lua_tostring(L, 1)` and `CHARACTER_MANAGER::FindPC(arg1)`.
4. Their actual lookup therefore expects a character-name string, but a normal non-numeric player name is rejected by the numeric gate first.

### Consequence
Quest code using the natural form `pc.is_blocked("PlayerName")` or `pc.is_friend("PlayerName")` cannot obtain the intended relation boolean for ordinary character names.

### Deferred validation
`MSG-T11`.
---

## BUG-MSG-012 — messenger receive buffer is still fixed at the legacy 24-character name limit

**Class:** client memory corruption / stack-buffer overflow  
**Reachability:** VERIFIED for friend, GM, block and mobile messenger name packets longer than 24 bytes.

### Proof
1. The canonical server character-name limit is `CHARACTER_NAME_MAX_LEN = 48`.
2. Server messenger send paths serialize `companion.size()` into a one-byte length and then send the full name bytes.
3. Client `CPythonNetworkStream::RecvMessenger()` still declares `char char_name[24 + 1]{}`.
4. Friend list/login/logout, GM list/login/logout, block list/login/logout and mobile branches call `Recv(length, char_name)` with the packet-provided length and then write `char_name[length] = 0`.
5. Those branches do not bound the received length to the 25-byte destination.

### Consequence
A valid character name longer than the legacy 24-byte limit can overwrite the client stack while messenger state is received. Depending on the packet and name length this can corrupt state or crash the client.

### Deferred validation
`MSG-T12`.

---

## BUG-MSG-013 — GM messenger query omits the valid WIZARD authority

**Class:** GM list completeness / authorization-model mismatch  
**Reachability:** VERIFIED from the server's canonical GM authority parser and messenger SQL.

### Proof
1. `EGMLevels` contains `GM_WIZARD` between `GM_LOW_WIZARD` and `GM_HIGH_WIZARD`.
2. DB admin loading explicitly maps `mAuthority = 'WIZARD'` to `GM_WIZARD`.
3. `MessengerManager::Login` builds the GM messenger list with a SQL predicate containing only `IMPLEMENTOR`, `HIGH_WIZARD`, `GOD` and `LOW_WIZARD`.
4. `WIZARD` is therefore a valid GM authority that can never enter `LoadGMList` through this query.

### Consequence
Staff accounts using the `WIZARD` authority are omitted from the GM messenger group and its presence display even though the server recognizes them as GMs.

### Deferred validation
`MSG-T13`.
---

## BUG-MSG-014 — name-based friend/block add bypasses the observer-mode restriction

**Class:** server authorization / path-parity defect  
**Reachability:** VERIFIED through the normal Messenger-window name input surfaces.

### Proof
1. `MESSENGER_SUBHEADER_CG_ADD_BY_VID` rejects the action when `ch->IsObserverMode()` is true.
2. `MESSENGER_SUBHEADER_CG_BLOCK_ADD_BY_VID` applies the same observer-mode rejection.
3. The corresponding `ADD_BY_NAME` and `BLOCK_ADD_BY_NAME` server branches do not check `IsObserverMode()`.
4. `uimessenger.py::OnAddFriend` normally exposes `SendMessengerAddByNamePacket(text)`.
5. `uimessenger.py::OnAddBlock` normally exposes `SendMessengerBlockAddByNamePacket(text)`.

### Consequence
Observer-mode restrictions are path-dependent: friend/block add actions rejected through a VID target can still be initiated by typing the same online player's name in the Messenger UI.

### Deferred validation
`MSG-T14`.



---

## BUG-MSG-015 — inverse watcher sets retain logged-out accounts indefinitely

**Class:** server lifecycle / stale-cache accumulation / memory-performance leak  
**Reachability:** VERIFIED for friend, block and synthetic GM messenger relations.

### Proof
1. Friend loading stores `m_Relation[account]` plus `m_InverseRelation[companion]`; block loading stores `m_BlockRelation[account]` plus `m_InverseBlockRelation[companion]`; GM loading stores `m_GMRelation[account]` plus `m_InverseGMRelation[gm]`.
2. `MessengerManager::Logout(account)` erases the account's outgoing friend/GM/block relations and removes the departing name from values in the corresponding outgoing maps.
3. Logout does not erase the departing account from the inverse sets representing that account as a watcher of its own friends, blocked names or GM entries.
4. Relation-specific remove helpers do prune inverse edges, but ordinary logout does not invoke them for the departing account's full outgoing relation set.
5. Repeated logins reinsert the same set members, so one account does not duplicate itself, but unique historical accounts remain resident for the process lifetime unless the underlying relation is explicitly removed.
6. Presence fanout iterates these inverse sets and attempts sends to stale/offline watcher names; send helpers later no-op when no local descriptor exists.

### Consequence
Friend, block and GM inverse watcher maps grow toward historical process-visible accounts rather than only currently useful watchers. Presence events repeatedly traverse stale names, increasing resident cache size and fanout work over long uptimes.

### Deferred validation
`MSG-T15`.

---
## BUG-MSG-016 — delayed old-core P2P logout can delete a newer channel/session presence

**Class:** multi-core ordering / lifecycle race  
**Reachability:** VERIFIED as a static cross-connection ordering race during channel/core handoff.

### Proof
1. A newly loaded character broadcasts `TPacketGGLogin` containing name, PID, map and channel.
2. `P2P_MANAGER::Login` looks up the CCI by name. If it already exists, the function updates that same CCI in place with the newest descriptor, map and channel.
3. The source core later broadcasts `TPacketGGLogout`, whose identity payload is the character name.
4. `CInputP2P::Logout` ignores the source descriptor for identity validation and calls `P2P_MANAGER::Logout(name)`.
5. `P2P_MANAGER::Logout(name)` resolves whichever CCI is currently stored under that name, invokes `MessengerManager::P2PLogout(name) -> Logout(name)`, erases the CCI maps and deletes the object.
6. The new login and old logout originate from different P2P peer connections; per-connection TCP ordering therefore does not impose a global order between them.
7. If an observer core receives LOGIN(new) first and delayed LOGOUT(old) second, the stale logout removes the **newly updated** CCI and tears down messenger presence for the still-online destination session.

### Consequence
A channel/core transition can make an affected process mark a still-online character offline, delete its current P2P CCI and run messenger relation/block logout cleanup against the newer session. This can compound `BUG-MSG-007` until another presence resynchronization occurs.

### Deferred validation
`MSG-T16`.
---

---

## BUG-MSG-017 — name-based messenger CG packets do not faithfully encode the configured 48-byte name range

**Class:** client/server protocol compatibility / string-boundary defect  
**Reachability:** VERIFIED for configured long character names on the normal Messenger name-entry and remove paths.

### Proof
1. The shared configured limit is `CHARACTER_NAME_MAX_LEN = 48`, and the player-create packet reserves `CHARACTER_NAME_MAX_LEN + 1` bytes for a character name.
2. Europe uses `check_name_alphabet`, which validates content/minimum length but does not impose a smaller maximum in the server validator.
3. `SendMessengerAddByNamePacket` and `SendMessengerBlockAddByNamePacket` allocate only `char szName[CHARACTER_NAME_MAX_LEN]`, copy at most `CHARACTER_NAME_MAX_LEN - 1`, force byte 47 to NUL, and send exactly 48 bytes. A 48-byte valid name therefore cannot be represented and is truncated to 47 bytes.
4. `SendMessengerRemovePacket`, `SendMessengerBlockRemovePacket`, and `SendMessengerBlockRemoveByVIDPacket` also allocate only 48 bytes and copy at most 47 bytes, but do **not** explicitly write a terminator into the final byte.
5. For 47-byte-or-longer names, the final transmitted byte can therefore remain uninitialized/stale while the server consumes the fixed 48-byte field and passes it to C-string handling via `strlcpy`.

### Consequence
Messenger name-based operations are not round-trip safe at the configured character-name boundary. Maximum-length names cannot be targeted correctly by add/block-add, while remove/unblock packets for long names can carry a non-terminated or contaminated final byte, causing lookup failure/state divergence and leaking one byte of client stack contents into the packet.

### Deferred validation
`MSG-T17`.
---

## BUG-MSG-018 — Battle Field friend-add restriction is bypassed by the VID path

**Class:** server authorization / path-parity defect  
**Reachability:** VERIFIED through the normal target-board friend action while `ENABLE_BATTLE_FIELD` is enabled.

### Proof
1. `ENABLE_BATTLE_FIELD` is enabled in the shared build defines.
2. `MESSENGER_SUBHEADER_CG_ADD_BY_NAME` explicitly rejects friend creation when `CBattleField::Instance().IsBattleZoneMapIndex(ch->GetMapIndex())` is true.
3. `MESSENGER_SUBHEADER_CG_ADD_BY_VID` contains no equivalent Battle Field map check.
4. `uitarget.py::OnAppendToMessenger` is the normal target-board friend action and sends `SendMessengerAddByVIDPacket(self.vid)`.
5. The target-board friend-button flow contains no Battle Field-specific gate before that send path.

### Consequence
The intended “cannot add friends in Battle Field” server restriction depends on which client surface is used. Typing a name is rejected, while targeting the same visible player and using the friend button can reach `RequestToAdd` through the unguarded VID branch.

### Deferred validation
`MSG-T18`.
---

---

## BUG-MSG-019 — pending friend authorization does not revalidate messenger block state

**Class:** TOCTOU / social authorization revalidation defect  
**Reachability:** VERIFIED through the normal friend-request acceptance flow.

### Proof
1. Both friend-request creation routes check messenger block state before calling `MessengerManager::RequestToAdd`.
2. `RequestToAdd` creates a CRC-based pending token in `m_set_requestToAdd` and sends `messenger_auth <requester>` to the target.
3. While that token is pending, either side can establish a messenger block because the friendship has not yet been created.
4. Normal client acceptance sends `/messenger_auth y <requester>` and reaches `MessengerManager::AuthToAdd`.
5. `AuthToAdd` checks only that the pending token exists, erases it, then calls `AddToList` in both directions.
6. The acceptance path never re-checks `IsBlocked` in either direction.

### Consequence
A friend request that was valid when issued can still be accepted after either side blocks the other, creating a contradictory friend + block state through a normal UI sequence independently of BUG-MSG-001.

### Deferred validation
`MSG-T19`.
## BUG-MSG-020 — oversized friend/block lists overflow the 16-bit messenger packet size

**Class:** protocol framing / list-size overflow  
**Reachability:** VERIFIED for sufficiently large persistent friend or block relation sets; no relation-count cap is enforced in the mapped manager paths.

### Proof
1. `TPacketGCMessenger::size` is a `uint16_t`, so one messenger packet can declare at most 65,535 bytes.
2. `MessengerManager::SendList` and `SendBlockList` build the complete relation list in a `TEMP_BUFFER(128 * 1024)`.
3. Every entry appends its connected/length metadata plus the full character-name bytes, with no cumulative packet-size check.
4. No relation-count limit is enforced in the mapped friend/block creation paths.
5. After construction, the code performs `pack.size += buf.size()`; payloads beyond the 16-bit range truncate/wrap the advertised size.
6. The server still transmits the complete buffer with `d->Packet(buf.read_peek(), buf.size())`.
7. Client `RecvMessenger` derives the list payload length from `p.size`, so bytes beyond the wrapped advertised size remain in the stream and can be interpreted as later packet framing.

### Consequence
A sufficiently large friend or block list can desynchronize the client network stream at list delivery, causing malformed subsequent packets, disconnects or client instability.

### Deferred validation
`MSG-T20`.

---

## BUG-MSG-021 — GM-to-GM block authorization differs between VID and name paths

**Class:** authorization path-parity defect  
**Reachability:** VERIFIED for a GM requester targeting another GM when `test_server == false`.

### Proof
1. The block-by-VID branch rejects a GM target only when the requester is a normal player:
   `ch->GetGMLevel() == GM_PLAYER && ch_companion->GetGMLevel() != GM_PLAYER && !test_server`.
2. A GM requester can therefore pass that check when blocking another GM by VID.
3. The block-by-name branch instead rejects whenever `gm_get_level(name) != GM_PLAYER && !test_server`, without considering the requester's GM level.
4. The target-board block button exposes the VID route, while the Messenger block-name dialog exposes the name route.
5. `AddToBlockList` has no later common authorization check that reconciles the two policies.

### Consequence
The same GM-to-GM block operation is allowed through the target/VID surface but denied through the name-entry surface solely because the protocol path differs.

### Deferred validation
`MSG-T21`.
