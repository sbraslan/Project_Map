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

## MSG-T12 — long messenger name receive
After runtime is explicitly unlocked in an isolated test client:
1. use a valid character name longer than 24 bytes and within the configured 48-character limit;
2. trigger friend list/login/logout delivery;
3. repeat for block and, where available, GM/mobile messenger branches;
4. observe client stability and use memory diagnostics if enabled.

Bug signature: the packet length exceeds the 25-byte local `char_name` buffer in `RecvMessenger`, causing memory corruption/crash.

## MSG-T13 — WIZARD authority missing from GM messenger list
After runtime is explicitly unlocked:
1. configure a staff character with `mAuthority = 'WIZARD'` and verify the server recognizes it as `GM_WIZARD`;
2. log in another account that receives the GM messenger list;
3. compare visibility/presence of WIZARD with LOW_WIZARD, HIGH_WIZARD, GOD and IMPLEMENTOR entries.

Bug signature: the WIZARD staff character is absent from the GM messenger list because the login SQL does not select that authority.

## MSG-T14 — observer-mode name-path friend/block add
After runtime is explicitly unlocked:
1. enter observer mode with A while B is online;
2. verify target/VID friend and block add attempts are rejected;
3. use the Messenger window to type B's name and submit friend add;
4. repeat with block add;
5. inspect friend-request/block state and DB rows.

Bug signature: name-based friend/block actions proceed in observer mode while the VID-based equivalents are server-rejected.

## MSG-T15 — GM inverse watcher accumulation
After runtime is explicitly unlocked in an isolated environment:
1. choose a known GM entry G returned by the GM messenger query;
2. log in and out a sequence of unique normal accounts while observing the same game process;
3. inspect `m_GMRelation` and `m_InverseGMRelation[G]` after those accounts log out;
4. trigger G login/logout presence and measure/trace the watcher iteration.

Bug signature: each logged-out account disappears from its outgoing GM relation but remains in `m_InverseGMRelation[G]`, so the inverse set and fanout work grow with historical unique accounts.

## MSG-T16 — stale old-core logout after newer channel login
After runtime is explicitly unlocked in a controlled multi-core environment:
1. keep an observer core C connected to both source core A and destination core B;
2. move player P from A to B;
3. delay/reorder delivery to C so C processes B's `GG_LOGIN(P)` before A's old `GG_LOGOUT(P)`;
4. inspect C's P2P CCI and MessengerManager login/relation state after both packets.

Bug signature: the delayed old logout resolves P only by name, deletes the CCI already updated to B and invokes messenger logout for the still-online destination session.

## MSG-T17 — configured long-name CG messenger round trip
After runtime is explicitly unlocked in an isolated test environment:
1. create/use valid 47-byte and 48-byte character names under the configured name limit;
2. from the Messenger name dialog, attempt friend add and block add against those names;
3. establish matching relations by another route where needed, then exercise friend remove, block remove and unblock-by-VID;
4. capture the 48-byte CG name field and compare it byte-for-byte with the intended character name and server lookup result.

Bug signature: 48-byte names are truncated by add/block-add to 47 bytes, while 47-byte-or-longer remove/unblock fields can transmit a non-terminated/stale final byte and fail to address the intended relation.


## MSG-T14 — observer-mode name-path bypass
After runtime is explicitly unlocked:
1. place A in observer mode and keep B online;
2. verify A cannot add/block B through the VID/target-board path;
3. open the Messenger window and type B's name into Add Friend / Add Block;
4. inspect the request/block state on both server and client.

Bug signature: the name-based path succeeds while the VID-based path is rejected solely because A is in observer mode.

## MSG-T18 — Battle Field friend-add VID parity
After runtime is explicitly unlocked:
1. enter a Battle Field map with A and visible player B;
2. verify Messenger-window add-by-name against B is rejected by the Battle Field restriction;
3. target B and use the target-board friend button;
4. observe whether B receives `messenger_auth` and whether the relation can be completed.

Bug signature: add-by-name is rejected on the Battle Field map while the target-board VID route creates the friend request.

Do not run any MSG test while the project execution lock is active.
