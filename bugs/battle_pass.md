# battle pass — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

### BUG-BPASS-001 — mission update packet sends uninitialized bMissionType
- Statik durum: **doğrulandı**
- Sınıf: protocol correctness / uninitialized-data leak

`TPacketGCExtBattlePassMissionUpdate` contains `bMissionType`.
Character update/set code creates non-zero-initialized packet and assigns header/passType/missionIndex/newProgress but not missionType.
Client reads missionType and uses it in `HaveMission(...)`/UI selection.

### BUG-BPASS-002 — SetExtBattlePassMissionProgress can re-award an already completed mission
- Statik durum: **doğrulandı; caller reachability audit açık**
- Sınıf: reward duplication

Existing matched mission is forcibly changed to `bCompleted = 0`, then value is overwritten.
If new value is at/above threshold, code marks completed and calls `BattlePassRewardMission` again.
Any legitimate/replayable caller that sets a completed mission can duplicate mission reward.

### BUG-BPASS-003 — BattlePassRequestOpen uses dangling season_name pointer
- Statik durum: **doğrulandı**
- Sınıf: use-after-lifetime / undefined behavior

Inside each pass block:
a local `std::string BattlePassName` is created, `season_name = BattlePassName.c_str()`, then the string is destroyed at block end.
The pointer is used afterward to fill the packet.

### BUG-BPASS-004 — unbounded strcpy into season-name packet
- Statik durum: **doğrulandı**
- Sınıf: stack overwrite / config-trust

`strcpy(packet.szSeasonName, season_name)` has no destination-size enforcement.
Battle-pass name comes from config loader.

### BUG-BPASS-005 — final reward path dereferences MYSQL_ROW without zero-row check
- Statik durum: **doğrulandı**
- Sınıf: server crash on inconsistent persistence state

After SELECT from `player.battlepass_playerindex`, code checks only SQL errno.
It calls `mysql_fetch_row` then immediately reads `row[0]`.
Missing registration row can therefore null-dereference.

### BUG-BPASS-006 — Event Manager cache arrays are not initialized
- Statik durum: **doğrulandı**
- Sınıf: uninitialized state / season selection

`CBattlePassManager` constructor initializes scalar active IDs/times but not:
- `m_dwActiveBattlePassID[3]`
- `m_dwBattlePassStartTime[3]`
- `m_dwBattlePassEndTime[3]`

The manager is an automatic object in main, and `InitializeBattlePass()` calls `CheckBattlePassTimes()`, which reads these arrays under ENABLE_EVENT_MANAGER.

### BUG-BPASS-007 — Event Manager stores boolean state as battle-pass ID
- Statik durum: **doğrulandı for configured IDs != 1**
- Sınıf: season lifecycle / wrong identity

`BattlePassData(const TEventTable*, uint8_t bType, bool bState)` calls:
`SetBattlePassID(bState, bType)`.
The setter stores that uint32 directly as active ID, so start state becomes ID 1 and stop becomes 0.
A configured season whose real battle-pass ID is not 1 cannot be represented through this path.

## Battle Pass — persistence/lifecycle completion

### BUG-BPASS-008 — mission reward durability is split from mission progress durability
- Statik durum: **doğrulandı**
- Sınıf: crash consistency / repeat reward

Mission completion immediately calls `BattlePassRewardMission`, which grants reward items using `AutoGiveItem`.
`CItem::AddToCharacter` calls `Save()`, placing the item into ITEM_MANAGER delayed-save state, and `ITEM_MANAGER::Update()` can persist it while the session is still running.

Battle Pass mission state is different: dirty `TPlayerExtBattlePassMission` records are sent to DB only from CHARACTER disconnect/logout and saved by DB as `REPLACE INTO battlepass_missions`.

Therefore this ordering exists:
1. mission becomes completed in GAME RAM;
2. reward item is granted;
3. item can become durable in DB;
4. battlepass_missions completion is still only RAM;
5. GAME crashes before clean logout;
6. next login reloads old mission progress and the mission can complete/reward again.

This needs fault injection for reproduction timing, but the persistence split itself is statically confirmed.

### BUG-BPASS-009 — final pass completion commits before final reward durability
- Statik durum: **doğrulandı**
- Sınıf: non-atomic claim / reward loss

`BattlePassRequestReward`:
1. verifies all missions;
2. SELECTs `battlepass_completed`;
3. executes synchronous UPDATE setting `battlepass_completed=1`;
4. only afterward calls `BattlePassReward`;
5. reward items use `AutoGiveItem` / delayed item save.

A crash or grant/save failure after step 3 and before reward items become durable leaves the DB claiming the final reward was consumed, so retry is rejected and the player can permanently lose the final reward.

### BUG-BPASS-010 — mission heap objects are never released
- Statik durum: **doğrulandı**
- Sınıf: memory leak

`LoadExtBattlePass`, normal Update and manual Set allocate `new TPlayerExtBattlePassMission` and store raw pointers in `m_listExtBattlePass`.
The CHARACTER initialization path only calls `m_listExtBattlePass.clear()`; disconnect iterates/saves the pointers but does not delete them.
No ownership cleanup for these allocations exists in the mapped source.
Character churn therefore leaks memory proportional to loaded/created mission rows.

### BUG-BPASS-011 — Battle Pass ranking cooldown timestamp is uninitialized
- Statik durum: **doğrulandı**
- Sınıf: uninitialized state / incorrect rate limiting

CHARACTER initialization sets `m_dwLastReciveExtBattlePassInfoTime = 0` but never initializes `m_dwLastExtBattlePassOpenRankingTime`.
The first ranking request reads it before the first setter call:
`if (get_dword_time() < GetLastReciveExtBattlePassOpenRanking())`.
An indeterminate value can incorrectly block the first ranking request or produce nonsensical remaining-time output.

### BUG-BPASS-006 impact refinement
Boot ordering confirms Event Manager table initialization occurs before `CBattlePassManager::InitializeBattlePass()`, but Event Manager initialization only builds event queues; it does not populate Battle Pass cache arrays.
`InitializeBattlePass()` then calls `CheckBattlePassTimes()`, so the uninitialized cache arrays remain a real first-boot read path under ENABLE_EVENT_MANAGER.

### BUG-BPASS-007 impact refinement
Current `Project_Game/share/locale/europe/battlepass/{normal,premium,event}.txt` all use `BattlePassID 1`.
Therefore bool `bState` accidentally equals the current ID while active.
This masks the defect today; any future season configured with ID 2+ will still be represented as active ID 1.

## Achievement System — initial canonical bugs
