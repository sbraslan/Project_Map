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

## Open candidates / not promoted yet
- add-by-name friend path differs from add-by-VID observer validation; gameplay impact not yet closed.
- remove-all and inverse-only relation cleanup semantics need symmetry audit.
- `OnBlockLogin` lacks the handler-null guard used by nearby callbacks; safety of the Python-call helper has not yet been proven.


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
Persistent relationships in SQL diverge from the live server cache after the companion logs out. Most critically, if A blocked B, B logging out and reconnecting can make `IsBlocked(A,B)` false until A itself reloads its list. This creates a same-core block-enforcement bypass in addition to the separate cross-core whisper bypass in BUG-MSG-002. Friend membership checks/duplicate prevention can also observe a false-negative cache.

### Deferred validation
`MSG-T07`.
