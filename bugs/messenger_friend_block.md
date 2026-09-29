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
