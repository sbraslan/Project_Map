# Messenger / Friend / Block — Deferred Runtime Tests

**Execution status:** LOCKED / NOT RUN  
**Global first future live gate:** `DUNGEON-T09`

## MSG-T01 — friend and block coexistence
After runtime is explicitly unlocked:
1. make A and B mutual friends;
2. from A, block B through name and VID surfaces separately;
3. inspect friend list, block list, DB rows and messages.

Bug signature: B remains a friend while also being added to A's block list; the intended remove-friend validation never triggers.

## MSG-T02 — cross-core whisper block
After runtime is explicitly unlocked:
1. block B from A;
2. verify same-process whisper rejection;
3. place A and B on different cores/channels with P2P relay;
4. attempt whispers in both directions.

Bug signature: remote/P2P whisper succeeds while local whisper is blocked.

## MSG-T03 — delayed DB callback after logout
After runtime is explicitly unlocked in an isolated environment:
1. delay messenger list SQL completion;
2. enter game and immediately disconnect before callbacks return;
3. release callbacks;
4. inspect relation/inverse maps and friend presence notifications.

Bug signature: callback repopulates logged-out account state and/or emits a false login notification.

## MSG-T04 — online blocked-user rendering
After runtime is explicitly unlocked:
1. block a user who is online;
2. force/observe `MESSENGER_SUBHEADER_GC_BLOCK_LOGIN`;
3. inspect the block group UI state.

Bug signature: entry is displayed offline because client routes block login to `OnLogout`.

## MSG-T05 — block/GM cache leakage across sessions
After runtime is explicitly unlocked:
1. log in with account/character A containing block/GM entries;
2. return to login/select without terminating the client;
3. enter with account/character B lacking those entries;
4. inspect `IsBlockFriendByName` and messenger groups.

Bug signature: A's block/GM names persist into B's session.

## MSG-T06 — stale friend authorization
After runtime is explicitly unlocked:
1. A sends a friend request to B;
2. B disconnects without accepting or denying;
3. wait beyond the normal UI lifetime and reconnect B;
4. issue the matching `/messenger_auth y A` response;
5. inspect DB and both friend lists.

Bug signature: the old request is still accepted and creates the relationship.

## MSG-T07 — block persistence across target logout/relogin
After runtime is explicitly unlocked:
1. keep A online and have A block B;
2. verify `IsBlocked(A,B)` behavior through a blocked interaction;
3. log B out while A stays online;
4. reconnect B without reconnecting A;
5. repeat the blocked interaction on the same core.

Bug signature: A's server-side outgoing block entry for B was erased at B logout and is not restored by B's reload, so the interaction is no longer blocked.

## MSG-T08 — blocked party-request UI bypass
After runtime is explicitly unlocked:
1. have B join or lead an existing party so A's target board shows “request to join party”;
2. establish a messenger block between A and B in either direction;
3. verify the direct party-invite path is rejected;
4. from A's target board, use the request-to-join-party button;
5. observe B's party-request UI/event.

Bug signature: the `/party_request <vid>` route reaches B despite the messenger block.

## MSG-T09 — unblock confirmation after target disappears
After runtime is explicitly unlocked:
1. block B from A;
2. target B and open the unblock confirmation dialog;
3. before accepting, make B leave the visible instance through logout, warp, channel/map change or range removal;
4. accept the unblock dialog.

Bug signature: `GetInstancePtr(vid)` returns null and the client dereferences it in `SendMessengerBlockRemoveByVIDPacket`, producing a client crash.

## MSG-T10 — block change during pending party/guild invite
After runtime is explicitly unlocked:
1. A sends B a party invite;
2. before B accepts, establish a messenger block between A and B;
3. accept the original pending party invite;
4. repeat the sequence with a guild invite.

Bug signature: the pre-existing invite still completes although a newly-created invite would now be rejected by messenger block.

## MSG-T11 — Lua friend/block name argument
After runtime is explicitly unlocked in a test quest:
1. keep two normally named characters online;
2. call `pc.is_blocked("OtherPlayer")` and `pc.is_friend("OtherPlayer")`;
3. compare the Lua return values with the actual messenger relations and server error log.

Bug signature: the helper takes the numeric-argument error path / returns no intended boolean for a normal player-name string.
Do not run any MSG test while the project execution lock is active.
