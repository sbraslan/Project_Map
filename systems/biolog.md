# biolog

**Status:** STATIC COMPLETE

> Canonical subsystem history split from legacy `00_PROGRESS.md`. Read this file only when this subsystem is active or explicitly revisited.

## Checkpoint — Biolog System STATIC COMPLETE

**Tarih:** 2026-09-26

Achievement STATIC COMPLETE sonrasında Biolog System client -> GAME -> quest -> player persistence -> DB proto zinciri kapatıldı.

### Canonical roots
- Server runtime: `game/src/BiologSystemManager.cpp/.h`
- Player state: `char.h/.cpp`
- GAME packet entry: `input_main.cpp::BiologManager`
- DB proto load: `db/src/ClientManagerBoot.cpp::{InitializeBiologMissions,InitializeBiologRewards}`
- DB player persistence: `ClientManagerPlayer.cpp`
- Client C++: `PythonBiologManager* + PythonNetworkStreamPhaseGame.cpp`
- Binary UI: `root/uibiologmanager.py`
- Quest bridge: `questlua_pc.cpp::pc_biolog_*`
- Runtime quest list: `Project_Game/share/locale/europe/quest/quest_list`

### Main submission flow
Expanded taskbar
-> `biologmgr.SendPacket(OPEN)`
-> CG 128 / OPEN
-> `CInputMain::BiologManager`
-> `CBiologSystem::SendBiologInformation`
-> GC 146 + `TPacketGCBiologManagerInfo`
-> client cache/UI.

Submit:
`SendPacketItem`
-> CG SEND
-> current mission lookup
-> level check
-> required-item count
-> cooldown
-> optional Researcher Elixir chance override
-> `RemoveSpecifyItem(1)`
-> RNG chance
-> collected count update
-> cooldown/reminder update
-> GC refresh.

### Player persistence
Player-table fields:
- biolog_mission
- biolog_collected
- biolog_cooldown_reminder
- biolog_cooldown

GAME:
`CreatePlayerProto` copies the four fields.
DB load/save uses the same four columns.

Mutation is not synchronously persisted by Biolog setters; it relies on normal CHARACTER save flow (default save event 120 seconds), while required item mutations use normal item delayed-save/destroy paths.

### Proto/config source
DB process loads:
- `biolog_missions`
- `biolog_rewards`

and sends their packed proto tables in the DB boot payload to each GAME process.

Actual table row contents are not present in the Git repositories, so value-level mission/reward validation remains a runtime/database-data dependency.

### Completion / quest bridge
When collected count reaches required count:
- sub-mission path sets `biolog_manager.can_do_sub_mission_<mission>`
- Seon-Pyeong selection path sets `biolog_manager.can_select_reward_<mission>`
- UI changes to the completion button and calls
  `SendRequestEventQuest("biolog_manager")`.

However current `Project_Game`:
- has no quest/object named `biolog_manager`;
- quest_list contains only legacy `collect_quest_lv30...94` biolog quests;
- those legacy quests do not call any `pc.biolog_*` APIs.

Therefore the new C++ manager has no mapped current quest consumer for its completion flags / reward / mission-advance bridge.

### Reward helper
Quest Lua exposes:
- biolog_has_mission
- biolog_get/set_mission
- biolog_get/set_cooldown
- biolog_get mission/sub-mission item
- biolog_get_reward_item
- biolog_set_reward_bonus.

`biolog_set_reward_bonus` loops every configured reward bonus and adds `AFFECT_COLLECT` with `bOverride=false`.

### Active verified bugs
- BUG-BIO-001: missing `biolog_manager` quest bridge leaves new system completion/reward/mission advance disconnected.
- BUG-BIO-002: TIMER variable payload length accounting can read beyond currently received packet bytes.
- BUG-BIO-003: `m_BiologReminderEventState` is not initialized before first reminder setter reads it.
- BUG-BIO-004: reward apply_type is not range validated; zero/invalid values can underflow/OOB `aApplyInfo`.
- BUG-BIO-005: item consumption/progress/cooldown persistence is non-atomic.
- BUG-BIO-006: repeated `pc.biolog_set_reward_bonus()` can stack permanent biolog affects.
- BUG-BIO-007: crafted SEND after required count is complete can consume Researcher Elixir affect without submitting an item.
- OBS-BIO-001: client reward-bonus Python getter accepts arbitrary index and directly indexes fixed arrays.
- OBS-BIO-002: sequence framing would be inconsistent if ENABLE_SEQUENCE_SYSTEM were re-enabled; current server/client flags are both disabled.
- OBS-BIO-003: empty biolog proto tables still reach boot encoding via `&vector[0]` with size zero, which is undefined C++ access.

### ABI classification
Current build is compatible:
- GAME Makefile uses `-m32`
- client UserInterface project is Win32 only
- packed `long/time_t` fields therefore have matching 32-bit widths.

### Status
**Biolog System: STATIC COMPLETE.**

Remaining work is runtime/fault-injection plus inspection of actual live `biolog_missions/biolog_rewards` DB rows.

### Next static subsystem
**Hunting System** selected next.

## Related
- Bugs: `../bugs/biolog.md`
- Runtime tests: `../tests/biolog.md`
- Full legacy archive: `../archive/00_PROGRESS.md`
