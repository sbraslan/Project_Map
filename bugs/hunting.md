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
For a character whose player level satisfies that hunting level, later select/open/update paths index Hunting mission/reward tables using level 91 with no terminal guard.

This is a normal-progression boundary bug, not only a modified-client path.
