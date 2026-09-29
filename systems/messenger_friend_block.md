# Messenger / Friend / Block — Static Mapping

**Status:** ACTIVE / SERVER ENTRYPOINT PASS 1 COMPLETE  
**Mode:** detection / mapping only  
**Source repositories:** read-only  
**Runtime execution:** locked

## Scope opened
- `Project_ServerSRC/game/src/input_main.cpp`
- `Project_ServerSRC/game/src/messenger_manager.cpp`
- `Project_ServerSRC/game/src/messenger_manager.h`
- `Project_ServerSRC/game/src/input_p2p.cpp`

## Confirmed server flow
Friend add:
`CG_MESSENGER ADD_BY_VID|ADD_BY_NAME -> CInputMain::Messenger -> RequestToAdd -> AuthToAdd -> AddToList -> SQL messenger_list + P2P`

Block add:
`CG_MESSENGER BLOCK_ADD_BY_VID|BLOCK_ADD_BY_NAME -> CInputMain::Messenger -> AddToBlockList -> SQL messenger_block_list + P2P`

Block remove:
`CG_MESSENGER BLOCK_REMOVE_BLOCK -> RemoveFromBlockList -> SQL delete + P2P`

## Verified defect
See `bugs/messenger_friend_block.md` — `BUG-MSG-001`.

## Exact next cursor
1. Trace P2P propagation and login/logout reconstruction for simultaneous friend+block state.
2. Trace client packet/UI entry points for both block-add variants.
3. Check whisper/shout/party/guild behavior when both relations coexist.
4. Check packet-size return mismatches in block-add-by-VID branches before promotion.
5. No runtime execution.


## P2P / login / logout pass 2

### Global propagation
- Local player entry calls `MessengerManager::Login(name)`.
- Remote channel entry reaches `P2P_MANAGER::Login`, which calls `MessengerManager::P2PLogin(name)` and therefore the same `Login(name)`.
- `Login(name)` independently queries `messenger_list` and `messenger_block_list`.
- Block add/remove is broadcast with `HEADER_GG_MESSENGER_BLOCK_ADD/REMOVE`; peers directly mutate their in-memory block relation via `__AddToBlockList/__RemoveFromBlockList`.

Therefore BUG-MSG-001 is not confined to one channel/process. No normalization removes the simultaneous friend+block state, and the actor's relog reloads both persistent rows independently.

### Logout reconstruction defect
`MessengerManager::Logout(target)`:
1. notifies `m_InverseBlockRelation[target]`,
2. iterates every `m_BlockRelation` set and erases `target`,
3. erases `m_BlockRelation[target]`.

The database row for another player blocking `target` is not deleted.

When `target` logs back in, `LoadBlockList(target)` loads only rows with `account=target`. It does not rebuild rows where `companion=target`.

Result: a still-online blocker may retain the block in DB/client presentation while `IsBlocked(blocker,target)` no longer has the corresponding in-memory entry. See BUG-MSG-002.

### Friend relation note
The same logout cleanup shape exists for `m_Relation`. Because friendship is stored bidirectionally, target relog restores only target's outgoing row; the other still-online owner's outgoing in-memory row is not rebuilt. This is recorded as a structural asymmetry, but no separate bug is promoted in this pass because current user-visible enforcement impact has not yet been proven.

## Exact next cursor
1. Trace client packet/UI entry points for block add by VID/name and block remove.
2. Check whisper/shout/party/guild behavior after BUG-MSG-002 desync.
3. Audit block-add-by-VID return-size mismatches.
4. No runtime execution.
