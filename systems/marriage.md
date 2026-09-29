# Marriage / Wedding — Static Map

**Status:** STATIC MAPPING IN PROGRESS / 2 VERIFIED BUGS  
**Phase:** Detection / Mapping Only  
**Opened:** 2026-09-29  
**Source/Game repositories:** READ-ONLY  
**Runtime:** LOCKED / NOT RUN

## Scope
Engagement creation, marriage persistence, wedding private-map orchestration, engaged-to-married transition, login/setup reconstruction, wedding termination and divorce/removal.

## Verified source roots
- `Project_ServerSRC/game/src/marriage.cpp`
- `Project_ServerSRC/game/src/marriage.h`
- `Project_ServerSRC/game/src/questlua_marriage.cpp`
- `Project_ServerSRC/game/src/wedding.cpp`
- `Project_ServerSRC/game/src/wedding.h`
- `Project_ServerSRC/game/src/input_db.cpp`
- `Project_ServerSRC/game/src/questlua_pc.cpp`
- `Project_ServerSRC/db/src/Marriage.cpp`
- `Project_ServerSRC/db/src/Marriage.h`
- `Project_ServerSRC/db/src/ClientManager.cpp`
- `Project_ServerSRC/common/tables.h`
- `Project_Game/share/locale/europe/quest/n_npc/marriage_manage.quest`
- `Project_Game/share/locale/europe/quest/quest_list`
- `Project_Game/share/locale/europe/map/metin2_map_wedding_01/*`
- deployed channel/core CONFIG files.

## Deployment
`MAP_WEDDING_01 = 81`.

Current tracked CONFIG ownership:
- `Project_Game/chan/ch2/core4/CONFIG` contains map 81.
- `Project_Game/chan/ch99/core99/CONFIG` also contains map 81.

The deployed quest list contains `n_npc/marriage_manage.quest`.

## Verified initial flows

### 1. Engagement creation
`marriage_manage.quest`
→ level/ring/sex/empire/dress/proximity checks
→ `confirm(u_vid,...)`
→ initiator pays 1,000,000 Yang
→ both 70301 engagement rings are converted to 70302
→ quest displays text and suspends at `wait()`
→ `marriage.engage_to(u_vid)`
→ `ALUA(marriage_engage_to)`
→ `marriage::CManager::RequestAdd`
→ `HEADER_GD_MARRIAGE_ADD`
→ DB `CClientManager::MarriageAdd`
→ DB `marriage::CManager::Add`
→ SQL `INSERT INTO marriage`
→ `HEADER_DG_MARRIAGE_ADD`
→ game `CInputDB::MarriageAdd`
→ game `marriage::CManager::Add`.

When both partners are online at game-side Add, the game core also emits `HEADER_GD_WEDDING_REQUEST`.

### BUG-MARR-001 — engagement resources commit before authoritative marriage creation
The deployed quest commits Yang and both ring conversions before the authoritative relation request, and then explicitly suspends at `wait()`.

If the quest is interrupted before it resumes, or if the target is no longer resolvable by the stored VID when `marriage.engage_to(u_vid)` finally runs, the Lua binding returns without `CManager::RequestAdd`. No rollback restores the already committed resources.

Promoted as `BUG-MARR-001`.

### 2. Wedding map orchestration
game `HEADER_GD_WEDDING_REQUEST`
→ DB `CClientManager::WeddingRequest`
→ unfiltered `ForwardPacket(HEADER_DG_WEDDING_REQUEST,...)`
→ all connected game peers
→ `CInputDB::WeddingRequest`
→ `WeddingManager::Request`
→ every receiving core for which `map_allow_find(81)` is true can create a private wedding map
→ loads `metin2_map_wedding_01/npc.txt`
→ `HEADER_GD_WEDDING_READY`
→ DB `CClientManager::WeddingReady`
→ `HEADER_DG_WEDDING_READY` + DB `ReadyWedding(...)`
→ 5-second DB start queue
→ `HEADER_DG_WEDDING_START`
→ game `CInputDB::WeddingStart`
→ `marriage::CManager::WeddingStart`
→ partner warp through the currently stored `pWeddingInfo->dwMapIndex`.

DB tracks running weddings and schedules the end boundary using `WEDDING_LENGTH = 60 * 60` seconds.

### BUG-MARR-002 — deployed map 81 ownership allows duplicate wedding-map producers
DB `WeddingRequest` broadcasts to every game peer with no channel filter. Current deployment has map 81 enabled on both ch2/core4 and ch99/core99.

Therefore both cores can independently execute `WeddingManager::Request`, create a private wedding map and send `HEADER_GD_WEDDING_READY` for the same pair.

DB `ReadyWedding` queues each READY without pair-level deduplication. Game `WeddingReady` retains only one `pWeddingInfo->dwMapIndex`, so later READY delivery can overwrite the previous map index. `TPacketWeddingStart` carries only PID1/PID2, not the selected map index.

Promoted as `BUG-MARR-002`.

### 3. Engaged → married transition
`marriage_manage.quest`
→ `marriage.set_to_marriage()`
→ `ALUA(marriage_set_to_marriage)`
→ `TMarriage::SetMarried`
→ `TMarriage::Save`
→ `CManager::RequestUpdate`
→ `HEADER_GD_MARRIAGE_UPDATE`
→ DB `marriage::CManager::Update`
→ SQL update of `love_point/is_married`
→ `HEADER_DG_MARRIAGE_UPDATE`
→ game `CInputDB::MarriageUpdate`.

### 4. Divorce / relation removal
`marriage_manage.quest`
→ `marriage.remove()`
→ `ALUA(marriage_remove)`
→ game `CManager::RequestRemove`
→ `HEADER_GD_MARRIAGE_REMOVE`
→ DB `marriage::CManager::Remove`
→ SQL `DELETE FROM marriage`
→ `HEADER_DG_MARRIAGE_REMOVE`
→ game `CInputDB::MarriageRemove`
→ game relation removal.

### 5. Game-core reconstruction
DB `marriage::CManager::OnSetup` replays persisted relations to a game peer using:
- `HEADER_DG_MARRIAGE_ADD`
- `HEADER_DG_MARRIAGE_UPDATE`
- running wedding `HEADER_DG_WEDDING_READY`
- running wedding `HEADER_DG_WEDDING_START`.

## DB packet map
- GD 70 / DG 150 — MARRIAGE_ADD
- GD 71 / DG 151 — MARRIAGE_UPDATE
- GD 72 / DG 152 — MARRIAGE_REMOVE
- GD 73 / DG 153 — WEDDING_REQUEST
- GD 74 / DG 154 — WEDDING_READY
- DG 155 — WEDDING_START
- GD 75 / DG 156 — WEDDING_END
- GD 116 / DG 159 — BREAK_MARRIAGE legacy path; closure still pending.

## Current audit cursor
Continue with:
1. legacy BREAK_MARRIAGE path;
2. mutual/unilateral divorce transaction ordering;
3. marriage login/logout + near-check/love-point lifecycle;
4. wedding map membership and teardown;
5. DB/game multi-core ordering after duplicate READY;
6. marriage unique-item bonus consumers;
7. Lua null/state assumptions against deployed callers.

## Runtime
No Marriage runtime test may be executed while the global execution lock is active. First future live gate remains `DUNGEON-T09`.
