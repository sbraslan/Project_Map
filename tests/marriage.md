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
