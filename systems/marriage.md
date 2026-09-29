# Marriage / Wedding — Static Map

**Status:** STATIC MAPPING IN PROGRESS  
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
- `Project_ServerSRC/db/src/Marriage.cpp`
- `Project_ServerSRC/db/src/Marriage.h`
- `Project_ServerSRC/db/src/ClientManager.cpp`
- `Project_Game/share/locale/europe/quest/n_npc/marriage_manage.quest`
- Wedding deployment: `Project_Game/share/locale/europe/map/metin2_map_wedding_01/*`

## Verified initial flows

### 1. Engagement creation
`marriage_manage.quest`
→ mutual confirmation / quest-side item+gold handling
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

### 2. Wedding map orchestration
game `HEADER_GD_WEDDING_REQUEST`
→ DB `CClientManager::WeddingRequest`
→ `HEADER_DG_WEDDING_REQUEST`
→ game `CInputDB::WeddingRequest`
→ `WeddingManager::Request`
→ allowed wedding-map core creates a private `WEDDING_MAP_INDEX`
→ loads `metin2_map_wedding_01/npc.txt`
→ `HEADER_GD_WEDDING_READY`
→ DB `CClientManager::WeddingReady`
→ `HEADER_DG_WEDDING_READY` + DB `ReadyWedding(...)`
→ 5-second DB start queue
→ `HEADER_DG_WEDDING_START`
→ game `CInputDB::WeddingStart`
→ `marriage::CManager::WeddingStart`
→ both partners warp to the private wedding map.

DB tracks the running wedding and schedules the end boundary using `WEDDING_LENGTH = 60 * 60` seconds.

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

## Current audit cursor
Continue with:
1. marriage login/logout + near-check/love-point event lifecycle;
2. wedding map membership and teardown;
3. quest precondition vs server-side invariant parity;
4. DB/game multi-core ordering and stale wedding/relation state;
5. divorce and wedding-end overlap;
6. client-visible lover command/affect behavior.

No bug is promoted until a full producer → validation → mutation → persistence/runtime consequence chain is proven.
