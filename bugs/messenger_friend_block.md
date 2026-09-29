# Messenger / Friend / Block — Bug Register

## BUG-MSG-001 — Friend check accidentally duplicates block check
**Status:** VERIFIED STATIC  
**Area:** server / messenger input validation  
**Source:** `Project_ServerSRC/game/src/input_main.cpp`

### Evidence
In both:
- `MESSENGER_SUBHEADER_CG_BLOCK_ADD_BY_VID`
- `MESSENGER_SUBHEADER_CG_BLOCK_ADD_BY_NAME`

the first guard emits the message equivalent of “Remove %s from your friends list to continue”, but calls:
`MessengerManager::Instance().IsBlocked(actor, target)`

The immediately following guard calls the same `IsBlocked(actor, target)` again and emits “already blocked”.

`MessengerManager` already exposes a separate friendship predicate:
`IsInList(account, companion)`.

### Static consequence
For a target who is already a friend but is not currently blocked:
- first duplicated `IsBlocked` => false
- second duplicated `IsBlocked` => false
- `AddToBlockList` executes
- friendship relation is not removed in this path

Therefore the server can create a simultaneous friend + block relation for the same directional pair.

### Expected guard shape
The first guard is strongly indicated to be the friendship predicate (`IsInList`), while the second remains `IsBlocked`.

### Runtime
Not executed. Runtime phase remains locked.


## BUG-MSG-002 — Target logout drops other users' active block enforcement from memory
**Status:** VERIFIED STATIC  
**Area:** server / messenger block lifecycle / P2P-login state reconstruction  
**Sources:** `Project_ServerSRC/game/src/messenger_manager.cpp`, `game/src/p2p.cpp`, `game/src/input_p2p.cpp`

### Evidence
For a directional block row `B -> A`:
1. B logs in: `LoadBlockList(B)` loads `A` into `m_BlockRelation[B]`.
2. A logs out: `MessengerManager::Logout(A)` erases `A` from every set in `m_BlockRelation`, including B's set.
3. The SQL row `B -> A` is not removed by logout.
4. A logs in again: local or P2P login calls `Login(A)`.
5. `LoadBlockList(A)` queries only `WHERE account='A'`; it never reloads rows where A is the companion.
6. `m_InverseBlockRelation[A]` can still notify B that A is online, but it does not repopulate `m_BlockRelation[B]`.
7. `IsBlocked(B,A)` therefore returns false until B's own block list is reloaded or another block-add mutation occurs.

### Consequence
The persistent block can remain represented in DB and client status while server-side checks backed by `IsBlocked(B,A)` lose the relation after the blocked target cycles offline -> online. This can disable block enforcement for paths that trust `IsBlocked`.

### Scope
Cross-channel relevant: P2P login/logout funnels into the same `MessengerManager::Login/Logout` lifecycle.

### Runtime
Not executed. Runtime phase remains locked.
