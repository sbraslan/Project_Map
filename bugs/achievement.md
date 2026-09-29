# achievement — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

### BUG-ACH-001 — completion reward is durable before achievement completion state
- Statik durum: **doğrulandı**
- Sınıf: crash consistency / repeat reward

`FinishAchievement` mutates only the in-memory player achievement map, sends client packets, then immediately calls `RewardPlayer`.
Rewards can be items (`AutoGiveItem`), gold, titles or achievement points.

The complete achievement/progress/points/title map is sent to DB only from `CAchievementSystem::OnLogout`.
DB then stores it in `CAchievementCache` for later flush.

Item rewards can independently become durable through normal ITEM_MANAGER delayed saves while the achievement completion remains only GAME RAM.
A hard GAME crash before clean logout can therefore reload the old unfinished state and allow the same achievement to finish/reward again.

### BUG-ACH-002 — achievement cache flush is non-transactional destructive rebuild
- Statik durum: **doğrulandı**
- Sınıf: persistence atomicity / data loss

`CAchievementCache::OnFlush` performs:
1. DELETE all `achievement_tasks` rows for pid and DELETE all `achievements` rows for pid;
2. REPLACE `achievement_data`;
3. INSERT every achievement row;
4. INSERT every unfinished task row.

These are separate DirectQuery calls with no transaction.
DB/process failure after the DELETE and before complete reconstruction can permanently leave a player with missing or partially rebuilt achievement/task state.

### BUG-ACH-003 — stale task ID can crash GetAchievementProgress for max_value achievements
- Statik durum: **doğrulandı**
- Sınıf: config-evolution / server crash

DB load preserves stored task IDs.
The login merge adds missing current tasks but does not remove obsolete task IDs from an existing achievement.

In `GetAchievementProgress`, when the current achievement has `max_value > 0`, code does:
`cTask = target_achievement->tasks.find(task.first)`
and immediately reads `cTask->second.type` without verifying `cTask != end()`.

If an XML update removes/renumbers a task while the DB still contains that old task ID, progress evaluation can dereference end() and crash the GAME core.

### BUG-ACH-004 — achievement shop can be opened remotely by client packet
- Statik durum: **doğrulandı**
- Sınıf: client trust boundary / interaction bypass

`CAchievementSystem::ProcessClientPackets` handles `HEADER_CG_OPEN_SHOP` by directly resolving:

`CShopManager::Instance().Get(104)`

and then calling `shop->AddGuest(player, 0, false)`.

There is no NPC VID validation, distance check, map check, quest state or proof that the character interacted with the intended achievement-shop NPC.
The nearby source comment explicitly says the player should have to open it from the NPC, but that enforcement is commented out.

Therefore a modified client can send the achievement OPEN_SHOP action from an arbitrary location and remotely enter shop 104.
The economic impact still depends on shop 104's actual currency/item configuration, which remains to be mapped.

### BUG-ACH-005 — Achievement Shop purchase can retain item while point debit rolls back after GAME crash
- Statik durum: **doğrulandı**
- Sınıf: crash consistency / currency rollback

Shop 104 uses `CoinType Achievement`.

`CShopEx::Buy` first verifies `GetAchievementPoints() >= price`, creates the item and reserves a valid inventory slot. It then calls:
`CAchievementSystem::ChangeAchievementPoints(ch, -dwPrice)`
and afterward adds the item and calls `ITEM_MANAGER::FlushDelayedSave(item)`.

Achievement points are not synchronously persisted by `ChangeAchievementPoints`; they are serialized with the rest of Achievement state only from `CAchievementSystem::OnLogout` and then stored through the DB achievement cache.

Crash ordering therefore exists:
1. points debited only in GAME memory;
2. purchased item added;
3. item explicitly flushed and becomes durable;
4. GAME crashes before clean Achievement logout save;
5. player reloads old Achievement points while retaining the purchased item.

This needs fault injection for deterministic reproduction, but the durability split is statically confirmed.

### BUG-ACH-006 — four configured task families have no mapped gameplay caller
- Statik durum: **doğrulandı**
- Sınıf: unreachable progression / dead achievement content

Current `achievements.xml` contains live tasks for:
- TYPE_SUMMON_MOUNT: 16 tasks
- TYPE_SPEND_SEARCH_SHOP: 5 tasks
- TYPE_SPEND_SHOP: 3 tasks
- TYPE_WITHDRAW: 5 tasks.

Caller audit found:
- `OnSummon` is called by PetSystem for TYPE_SUMMON_PET, but no mount/horse path calls it with TYPE_SUMMON_MOUNT.
- normal shop / ShopEx / shop manager paths do not call `OnGoldChange(...TYPE_SPEND_SHOP)`.
- mapped private-shop/search entry paths do not call TYPE_SPEND_SEARCH_SHOP.
- safebox/withdraw paths do not call TYPE_WITHDRAW.

The configured achievements in these families therefore cannot progress through the mapped gameplay implementation.

### BUG-ACH-007 — EXPLORE progression is only evaluated on login
- Statik durum: **doğrulandı**
- Sınıf: missing lifecycle hook / delayed achievement update

All current TYPE_EXPLORE tasks have max_value=1.

`CAchievementSystem::OnLogin` calls `OnVisitMap(player)`, but the mapped character movement/warp/input/dungeon paths contain no corresponding `OnVisitMap` call when the player actually enters another map.

A player who enters an exploration target during an existing session does not get immediate progress; the task is evaluated only after a later login while located on that map.

### BUG-ACH-008 — force-finish can re-grant an already completed achievement
- Statik durum: **doğrulandı**
- Sınıf: trusted admin/script correctness / repeat reward

Two force surfaces exist:
- `/force_finish_achievement` restricted to `GM_IMPLEMENTOR`;
- quest Lua `pc.finish_achievement(id)`.

Both call `CAchievementSystem::FinishAchievement`.

`FinishAchievement` does not check `IsAchievementFinished` or the existing task-0 completion marker before clearing/replacing the map and calling `RewardPlayer`.
Calling the force path repeatedly for the same achievement therefore re-grants its reward each time.

This is not a normal-player packet exploit in the mapped source; it is a GM/trusted-quest duplication hazard.


## Biolog System — canonical bugs
