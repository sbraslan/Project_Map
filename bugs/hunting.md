# hunting — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

### BUG-HUNT-001 — client-controlled mission type indexes Hunting tables without validation
- Statik durum: **doğrulandı**
- Sınıf: client trust / out-of-bounds table access

CG Hunting action 2 accepts `p->dValue` and writes it directly to:
`hunting_system.type`
with no check that it is 0 or 1.

It immediately calls `OpenHuntingWindowMain`, where the stored type is used in:
- `THuntingMissions[level][type]`
- `THuntingRewardItem[level][type][race]`.

Official UI sends only 0/1, but a modified client can supply an arbitrary uint32 value and drive out-of-range table reads.

### BUG-HUNT-002 — reward action can advance Hunting level without completing or caching a reward
- Statik durum: **doğrulandı**
- Sınıf: progression bypass

CG action 4 calls `ReciveHuntingRewards()` unconditionally.

That function does not require:
- `reward_cached == 1`,
- active mission,
- completed count,
- valid current type.

Even when all reward flags are zero, the function ends by:
- reward_cached=0
- is_active=-1
- type=-1
- count=0
- `level = level + 1`.

A modified client can therefore repeatedly send action 4 and skip Hunting progression levels without doing missions.

### BUG-HUNT-003 — gold reward is consumed when GOLD_MAX rejects the credit
- Statik durum: **doğrulandı**
- Sınıf: reward loss / missing commit check

If `reward_money != 0`, Hunting calls:
`PointChange(POINT_GOLD, reward_money, true)`
then always clears `hunting_system.reward_money`.

The POINT_GOLD branch returns without modifying gold when `GetGold()+amount >= GOLD_MAX`.
Hunting receives no success result and still clears the cached reward flag, permanently consuming the gold reward.

### BUG-HUNT-004 — completing mission 90 advances into unsupported level 91
- Statik durum: **doğrulandı**
- Sınıf: terminal-state / OOB progression

Active build defines:
`HUNTING_MISSION_COUNT 90`.

Random reward tables and level bands end at mission 90.

Every reward claim unconditionally executes:
`hunting_system.level = hunting_system.level + 1`.

Thus legitimate completion/claim of mission 90 stores level 91.
For a character whose player level is >=91, the later normal open flow reaches `OpenHuntingWindowSelect()`, which indexes Hunting mission/reward tables with level 91 and has no terminal guard.

The client-side completed-state UI is too late to protect this server access: the server must construct the packet first.

### BUG-HUNT-005 — item reward can persist before reward flags, allowing duplicate claim after GAME crash
- Statik durum: **doğrulandı**
- Sınıf: crash consistency / non-atomic reward claim / duplication

Hunting grants race/random items and then clears their quest flags, but the two sides use independent persistence paths.

Item side:
- `CItem::Save()` -> `ITEM_MANAGER::DelayedSave`;
- `ITEM_MANAGER::Update()` runs about every 5.08 seconds;
- item state is written through `HEADER_GD_ITEM_SAVE`.

Quest-flag side:
- `SetQuestFlag` -> `PC::SetFlag` only queues the change in `m_FlagSaveMap`;
- the normal character save event runs every 120 seconds;
- `SaveReal()` eventually calls `PC::Save()`, which writes `HEADER_GD_QUEST_SAVE`.

There is therefore a real persistence window where:
1. the newly granted reward item has already been stored in DB;
2. the old nonzero `reward_race` / `reward_rand` / `reward_cached` state is still the DB version;
3. GAME crashes before quest flags are saved.

After relog, the persisted item remains while the stale reward flags can allow the reward to be claimed again.

This is not one transactional commit and is independent of the client packet bypass in BUG-HUNT-002.

## Robustness findings not yet promoted to verified current-data bugs

### CreateItem null handling
`ReciveHuntingRewards()` does not check the return from `ITEM_MANAGER::CreateItem`.
`CreateItem` can return `nullptr` for a missing/invalid item proto or another creation failure, after which Hunting can dereference the null pointer during inventory/ground handling.

Current Hunting reward VNUM reachability still requires comparison against the actual server item-proto dataset before assigning a separate verified bug ID.

### Ground fallback result ignored
When inventory is full, Hunting calls `AddToGround(...)` but ignores its boolean return value, then starts ownership/destroy handling and clears reward flags.
If ground insertion fails, this can become reward loss. Normal-player reachability requires runtime/fault-injection confirmation.
