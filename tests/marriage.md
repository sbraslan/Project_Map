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


## MARR-T07 — EXP love-point progression across level boundary
After runtime is explicitly unlocked:
1. use a married level-25 character with spouse in the same map and valid near state;
2. gain a controlled amount of EXP and record love-point delta;
3. repeat at level 26 with the same EXP amount and relationship state;
4. optionally repeat at higher levels.

Expected bug signature: level 25 produces an EXP-derived update while level 26+ produces zero because the coefficient truncates before multiplication.

Covers `BUG-MARR-007`.

Do not run while the execution lock is active.


## MARR-T08 — cross-core marriage item sharing
After runtime is explicitly unlocked:
1. marry two characters and establish a known love-point percentage;
2. equip one tracked marriage bonus item (71069..71074) on spouse A only;
3. place both spouses on the same game core and record A/B bonus behavior;
4. move spouse B to a different game core without changing equipment or marriage state;
5. repeat the same relevant combat/EXP measurement on B.

Expected bug signature: B receives the advertised shared effect when both are represented on one process but loses it after cross-core separation, while A's item remains equipped.

Covers `BUG-MARR-008`.

Do not run while the execution lock is active.


## MARR-T09 — DB restart with pending engagement
After runtime is explicitly unlocked in an isolated environment:
1. create a valid engagement but do not complete `set_to_marriage`;
2. verify the `marriage` SQL row has `is_married=0`;
3. restart only the DB cache/server process;
4. inspect the row and both game-core relation states after reconnect/setup;
5. inspect player rings/Yang.

Expected bug signature: the SQL row is deleted at DB initialization and the engagement cannot be reconstructed/refunded.

Covers `BUG-MARR-009`.

## MARR-T10 — Marriage Fast monotonicity
After runtime is explicitly unlocked:
1. use a controlled marriage age and zero/known stored love_point;
2. record `GetMarriagePoint()` without Marriage Fast;
3. activate Marriage Fast and record again;
4. let/force the premium condition expire without changing marriage age/storage;
5. record the result again.

Expected bug signature: point value jumps upward under the current premium calculation and then drops when the premium condition becomes false.

Covers `BUG-MARR-010`.

Do not run these tests while the global execution lock is active.


## MARR-T11 — game-core restart during running wedding
After runtime is explicitly unlocked:
1. start a wedding and identify the core that owns its private map;
2. keep DB running and restart only that game core;
3. observe DB OnSetup replay of WEDDING_READY/START;
4. verify whether `WeddingManager::Find(savedMapIndex)` exists afterward;
5. let/send wedding END and inspect `pWeddingInfo` / `m_setWedding` cleanup.

Expected bug signature: relation-side wedding state is replayed without recreating the private WeddingMap, and END fails on the map-owning core.

Covers `BUG-MARR-011`.

## MARR-T12 — DB restart during running wedding
After runtime is explicitly unlocked:
1. start a wedding and confirm it is in DB `m_mapRunningWedding`;
2. restart only the DB process while game core/private map remains alive;
3. verify the running wedding is absent from reconstructed DB state;
4. wait beyond the original one-hour boundary or issue manual end;
5. inspect whether DG WEDDING_END is ever emitted.

Expected bug signature: timeout state is lost and manual end is rejected because DB no longer knows the running pair.

Covers `BUG-MARR-012`.

Do not run these tests while the global execution lock is active.


## MARR-T13 — unilateral divorce while spouses are on different cores
After runtime is explicitly unlocked:
1. place married spouses on maps hosted by different game cores;
2. on spouse A, execute the deployed unilateral divorce route;
3. confirm DB relation removal and DG MARRIAGE_REMOVE fanout;
4. observe both clients' Messenger family group and lover affect icon without relogging.

Expected bug signature: relation is removed server-side but neither client receives `lover_divorce`, leaving stale lover UI until a later reset.

Covers `BUG-MARR-013`.

Do not run this test while the global execution lock is active.


## MARR-T14 — divorce while DB wedding is still running
After runtime is explicitly unlocked:
1. create a marriage whose wedding is currently active;
2. leave the private wedding map using an ordinary stored recall/talisman destination;
3. satisfy the deployed divorce-time requirement and execute unilateral divorce;
4. verify the marriage SQL/relation is removed while DB still has the running wedding;
5. let the original wedding timeout fire;
6. trace DG WEDDING_END and game `CManager::WeddingEnd`;
7. inspect `WeddingManager::Find(privateMapIndex)` afterward.

Expected bug signature: game rejects WEDDING_END because the marriage relation is already absent, so `WeddingManager::End` is not invoked and the private map remains registered.

Covers `BUG-MARR-014`.

Do not run while the global execution lock is active.
