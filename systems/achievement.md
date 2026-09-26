# achievement

**Status:** STATIC COMPLETE

> Canonical subsystem history split from legacy `00_PROGRESS.md`. Read this file only when this subsystem is active or explicitly revisited.

## Checkpoint — Achievement System audit started

**Tarih:** 2026-09-26

Battle Pass STATIC COMPLETE sonrası Achievement System ana omurgası açıldı.

Mapped roots:
- Server: `game/src/AchievementSystem.cpp/.h`
- DB: `db/src/ClientManagerAchievement.cpp`, `db/src/Cache.cpp::CAchievementCache`
- Client: `UserInterface/PythonAchievement.cpp/.h`
- Binary UI: `root/uiachievementsystem.py`, `root/uiachievementwrapper.py`
- Runtime config: `Project_Game/share/locale/europe/achievements.xml` (current count: 179 achievements)

Initial flow:
gameplay hooks
-> `CAchievementSystem::On*`
-> per-character `TAchievementsMap`
-> `FinishAchievement`
-> notification/update
-> immediate `RewardPlayer`
-> logout serialization
-> `HEADER_GD_ACHIEVEMENT`
-> DB `CAchievementCache`
-> delayed cache flush/rebuild of achievement tables.

Initial verified bugs:
- BUG-ACH-001 reward is granted immediately but completion/progress is logout-persisted -> crash repeat-reward window.
- BUG-ACH-002 DB cache flush deletes/rebuilds multiple tables without transaction -> partial/lost state on failure.
- BUG-ACH-003 stale task IDs can dereference end iterator in max_value progress path after config evolution.

Status: **PARTIAL**.

Next:
1. map every gameplay `On*` caller and task type.
2. map client packet entry + shop/ranking/title trust boundaries.
3. audit force-finish/admin paths.
4. audit achievement shop currency/inventory interaction.
5. audit XML config constraints/reload behavior.

## Checkpoint — Achievement System second pass

**Tarih:** 2026-09-26

Achievement System PARTIAL audit continued from the canonical checkpoint.

### Newly closed areas
- client -> GAME achievement action boundary:
  - SELECT_TITLE
  - OPEN_SHOP
  - OPEN_RANKING
- title ownership validation path
- achievement reward execution
- DB load/cache/ranking path
- kill/death and character-update core logic
- current achievements.xml domain sanity

### Config snapshot
Current `Project_Game/share/locale/europe/achievements.xml` contains:
- 179 achievements
- 578 tasks
- 532 restriction entries
- 248 reward entries
- task types used: 1..32
- no task type outside TYPE_MAX_NUM
- no restriction type outside RESTRICTIONS_MAX_NUM
- reward types currently used: TITLE and ACHIEVEMENT_POINTS
- no duplicate achievement IDs
- no zero-max task entries

This means BUG-ACH-003 is primarily a config-evolution / stale-DB-state risk rather than a currently malformed XML row.

### New verified trust-boundary bug
- **BUG-ACH-004:** `HEADER_CG_OPEN_SHOP` opens shop ID 104 directly from a client packet with no NPC identity, proximity, map, quest or interaction validation in `CAchievementSystem::ProcessClientPackets`. The source even contains a commented note saying the shop should be opened from the NPC. A modified client can therefore open the achievement shop remotely.

### Additional observations
- title selection itself is server-side ownership checked against `GetAchievementTitles()`.
- ranking data is requested server->DB and returned from the DB-owned cached ranking; the client does not submit ranking contents.
- achievement rewards are still immediate while achievement map/points/title persistence remains logout/cache based (BUG-ACH-001).
- current XML count 179 fits the uint8 achievement-count field used by GC_load; this becomes a protocol/config limit if the config ever exceeds 255 entries.

### Status
Achievement System remains **PARTIAL**.

### Next
1. finish gameplay `On*` caller reachability audit across the source tree.
2. map achievement shop currency/debit path for shop 104.
3. inspect force-finish/admin command callers.
4. close XML reload/config-evolution behavior.
5. decide Achievement System STATIC COMPLETE and then move to the next unmapped subsystem.

## Checkpoint — Achievement System STATIC COMPLETE

**Tarih:** 2026-09-26

Achievement System final static pass completed through gameplay caller coverage, client trust boundaries, ShopEx currency/debit flow, DB persistence, force-finish surfaces and config lifecycle.

### Gameplay caller matrix
- TYPE_KILL / TYPE_DIE -> `char_battle.cpp::OnKill`
- TYPE_REACH_LEVEL / TYPE_REACH_PLAYTIME / TYPE_REACH_SPEED -> `char.cpp::OnCharacterUpdate`
- TYPE_SUMMON_PET -> `PetSystem.cpp::OnSummon`
- TYPE_ACTIVATE_TOGGLE -> `char_item.cpp::OnToggle`
- TYPE_FISH / TYPE_BURN / TYPE_USE_BURN -> `fishing.cpp` + `char_item.cpp::OnFishItem`
- TYPE_WIN_WARS -> `guild_manager.cpp::OnWinGuildWar`
- TYPE_DEAL_DAMAGE / TYPE_GET_DAMAGAE -> `char_skill.cpp::DamageDealt`
- TYPE_COLLECT / TYPE_COLLECT_ALIGNMENT / TYPE_COLLECT_GOLD -> `char_battle.cpp` + `char_item.cpp::Collect`
- TYPE_SPEND_UPGRADE -> `char.cpp::PayRefineFee -> OnGoldChange`
- TYPE_TRADE -> `exchange.cpp::OnTrade`
- TYPE_UPGRADE_9 / TYPE_UPGRADE_15 -> `char_item.cpp::OnUpgrade`
- TYPE_ADD_FRIEND -> `messenger_manager.cpp::OnSocial`
- TYPE_JOIN_GUILD -> `guild.cpp::OnSocial` plus login backfill when already in guild
- TYPE_WHISPER / TYPE_SHOUTS -> `input_main.cpp::OnSocial`
- TYPE_PARTY -> `party.cpp::OnSocial`
- TYPE_SKILL -> `char_skill.cpp::OnMasterSkill`
- TYPE_DUNGEON -> `dungeon.cpp::OnFinishDungeon`
- TYPE_EXPLORE -> only `OnLogin -> OnVisitMap`; no map-transition/warp hook found.

### Missing configured gameplay hooks
Current XML actively contains tasks for types with no mapped gameplay caller:
- TYPE_SUMMON_MOUNT: 13 tasks
- TYPE_SPEND_SEARCH_SHOP: 5 tasks
- TYPE_SPEND_SHOP: 3 tasks
- TYPE_WITHDRAW: 5 tasks

These task families cannot progress through the mapped live gameplay paths.

### Achievement Shop 104 closed
`shop_table_ex.txt`:
- Vnum 104
- CoinType Achievement
- items currently priced 10 / 2 / 5 achievement points.

Client action:
`HEADER_CG_OPEN_SHOP`
-> `ProcessClientPackets`
-> direct `Get(104)->AddGuest` with no NPC/distance authorization.

Purchase:
`CShopEx::Buy`
-> achievement-point balance check
-> CreateItem + inventory-space check
-> `ChangeAchievementPoints(-price)`
-> AddToCharacter
-> `FlushDelayedSave(item)`.

Achievement points remain part of logout-only achievement persistence, while the purchased item is immediately flushed. This creates a crash-consistency point-refund/item-retention window.

### Force-finish surfaces
- GM command `force_finish_achievement` is restricted to `GM_IMPLEMENTOR`.
- Quest Lua registers:
  - `pc.is_achievement_finished`
  - `pc.finish_achievement`
  - `pc.finish_achievement_task`
- `FinishAchievement` itself does not reject an already finished achievement and calls `RewardPlayer` again. Therefore repeated GM/Lua force-finish can duplicate rewards.
- `FinishAchievementTask` does stop when total progress is already 100%.

### Config lifecycle
`CAchievementSystem achievement` is initialized once in `main.cpp` and `achievement.Initialize()` loads `achievements.xml` during server boot.
No Achievement-specific runtime reload command was found in the mapped command set.
Config changes therefore take effect after restart; stale DB task IDs remain the BUG-ACH-003 migration risk.

### Final verified bug set additions
- BUG-ACH-005: Achievement Shop item durability vs point-debit durability split.
- BUG-ACH-006: configured task families with no gameplay caller.
- BUG-ACH-007: EXPLORE updates only at login, not when entering a map.
- BUG-ACH-008: repeated force-finish re-grants rewards.

### Status
**Achievement System: STATIC COMPLETE.**

Remaining work is runtime/fault-injection testing.

### Next static subsystem
**Biolog System** selected next:
- server root: `game/src/BiologSystemManager.cpp/.h`
- expected scope: client action -> mission state -> item submission -> cooldown/chance -> reward -> DB/player persistence.

## Related
- Bugs: `../bugs/achievement.md`
- Runtime tests: `../tests/achievement.md`
- Full legacy archive: `../archive/00_PROGRESS.md`
