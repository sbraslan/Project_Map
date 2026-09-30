# Event MiniGames — Static System Map

**Status:** STATIC MAPPING CLOSED  
**Mode:** detection / mapping only  
**Execution:** LOCKED / NOT RUN  
**Source snapshot:** pinned by `MAP_STATE.json`

## Canonical scope
This node owns the shared event/minigame service layer and its independently stateful game families:
- `CMiniGameManager`;
- Okey/Rumi card game;
- Catch King;
- FindM;
- Black & White (BNW);
- attendance event/reward flow;
- Easter/banner/event NPC orchestration;
- shared minigame packet routing, event flags, ranking/reward hooks.

## Deployment proof
- `ENABLE_MINI_GAME` is enabled in ServerSRC.
- `ENABLE_MINI_GAME_OKEY_NORMAL`, `ENABLE_MINI_GAME_CATCH_KING`, `ENABLE_MINI_GAME_FINDM` and `ENABLE_MINI_GAME_BNW` are enabled.
- `ENABLE_MINI_GAME_YUTNORI` exists in source but is disabled on the pinned snapshot and is therefore a dormant branch, not an active audit target.
- `main.cpp` constructs the singleton `CMiniGameManager` and runs minigame end checks.
- `input_main.cpp` routes dedicated CG packet families into Okey, Catch King, BNW, FindM and YutNori handlers.
- `packet.h` defines dedicated CG/GC minigame packet families.
- `CHARACTER` owns per-player minigame runtime state for Catch King, BNW, FindM, YutNori and Okey.
- BNW and other minigames include ranking/reward state and DB/event-flag interactions.
- Binary contains dedicated UIs for Catch King, FindM, Fish Event, Roulette, Rumi/Okey, YutNori and event overview surfaces.

## Lifecycle boundary
Owned here:
- minigame request validation;
- per-character game state;
- event enable/disable flags;
- minigame score/reward/ranking flows;
- event NPC/banner orchestration;
- minigame packet framing and subheader dispatch.

Not owned here:
- OX Event lifecycle (already CLOSED canonical node);
- generic quest/event engine;
- generic Shop, Inventory, Safebox, Dungeon and Ranking ownership;
- pure seasonal art/assets without stateful server behavior.

## Active branches
- Okey / Rumi: ACTIVE
- Catch King: ACTIVE
- FindM: ACTIVE
- Black & White: ACTIVE
- Attendance / Monster Back: ACTIVE when event flag/deployment enables it
- Easter/banner orchestration: ACTIVE infrastructure
- YutNori: DORMANT_COMPILE_DISABLED on pinned snapshot

## Audit cursor
1. Audit CG packet/subheader length validation and active-event gating for Okey/Catch King/FindM/BNW.
2. Audit per-character state initialization/reset/disconnect/relogin boundaries.
3. Audit reward/payment ordering, duplicate-claim/replay paths and inventory-space handling.
4. Audit DB ranking queries/writes and event-end transitions.
5. Audit attendance reward-vector/file loading and empty/malformed-data paths.
6. Close client/server packet parity and promote only source-proven defects.

## Early risk candidates (not yet promoted)
- Attendance info sends from `attendanceRewardVec[0]`; prove whether an empty reward vector is reachable before classifying.
- BNW random-opponent selection assumes at least one unused opponent card; prove state invariants before classifying.
- Recursive Okey card randomization is bounded by deck uniqueness in intended state; verify corrupted/desynchronized state handling before classifying.


## Closeout
- Client/server packet headers and active minigame packet families were checked for parity on the pinned snapshot.
- No additional source-proven parity defect was found.
- Final verified bugs: 4.
- Final test plans: 4.
- Lifecycle is now CLOSED and locked on this source snapshot.
