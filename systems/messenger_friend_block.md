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
