# Marriage / Wedding — Deferred Runtime Tests

**Status:** DOCUMENTED / EXECUTION LOCKED / NOT RUN  
**Global first future live gate:** DUNGEON-T09

## MARR-T01 — engagement interruption after committed resource mutation
After runtime is explicitly unlocked:
1. prepare two eligible characters with 70301 rings, required dress and sufficient Yang;
2. complete mutual confirmation;
3. reach the post-conversion dialogue/`wait()` state;
4. interrupt the quest before the initiator resumes to `marriage.engage_to(u_vid)`;
5. relog both characters;
6. inspect Yang, 70301/70302 counts and marriage/engagement state.

Expected bug signature: resources/rings changed but no marriage relation exists.

Covers `BUG-MARR-001`.

## MARR-T02 — duplicate map-81 wedding producers
After runtime is explicitly unlocked:
1. preserve current deployment where map 81 is enabled on ch2/core4 and ch99/core99;
2. create one valid engagement;
3. trace the single originating `HEADER_GD_WEDDING_REQUEST`;
4. record all peers receiving `HEADER_DG_WEDDING_REQUEST`;
5. record all private-map creations and `HEADER_GD_WEDDING_READY` messages;
6. record DB start-queue entries;
7. record final `pWeddingInfo->dwMapIndex` and cleanup of every created map.

Expected bug signature: both map-81 cores create/send READY for the same pair, causing duplicate scheduling or an orphaned/overwritten wedding-map lifecycle.

Covers `BUG-MARR-002`.

Do not run either test while the execution lock is active.


## MARR-T03 — exact mutual-divorce fee boundary
After runtime is explicitly unlocked:
1. give both spouses exactly 500,000 Yang and required marriage rings;
2. satisfy divorce-time/proximity conditions;
3. attempt mutual divorce;
4. compare with a 500,001 Yang control.

Expected bug signature: exactly 500,000 is rejected while 500,001 passes the money predicate.

Covers `BUG-MARR-003`.

## MARR-T04 — stale P2P logout after fresh core login
After runtime is explicitly unlocked in a multi-core isolated setup:
1. keep spouse B online on a third core;
2. move spouse A from core X to core Y;
3. arrange/trace ordering where B's core processes LOGIN(A@Y) before delayed LOGOUT(A@X);
4. observe CCI ownership and lover commands on both clients.

Expected bug signature: the delayed old logout deletes the fresh CCI and emits `lover_logout` after the already-processed fresh login, leaving lover UI offline.

Covers `BUG-MARR-004`.

## MARR-T05 — wedding saved-exit destination
After runtime is explicitly unlocked:
1. record a participant's pre-wedding map/coordinates;
2. enter the wedding through the normal flow;
3. end the wedding;
4. trace `SaveExitLocation`, `WarpEnd`, and `ExitToSavedLocation` field values;
5. record the attempted exit destination.

Expected bug signature: `m_posExit/m_lExitMapIndex` contain the real source location, but exit uses cleared/incorrect `m_posWarp/m_lWarpMapIndex`.

Covers `BUG-MARR-005`.

## MARR-T06 — same-core wedding membership teardown
After runtime is explicitly unlocked and after isolating the saved-location bug:
1. host wedding map 81 and the saved destination on the same game process;
2. enter the wedding and verify WeddingMap membership;
3. end the wedding and successfully warp the character to the saved same-core destination;
4. inspect `m_pWeddingMap` / WeddingMap member set;
5. wait for the 15-second teardown step.

Expected bug signature: the character remains in the WeddingMap set after warp and is destroyed/disconnected by `DestroyAll()`.

Covers `BUG-MARR-006`.

Do not run these tests while the execution lock is active.
