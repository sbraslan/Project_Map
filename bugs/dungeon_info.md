# dungeon info — Bug Registry

> Canonical split from legacy `08_BUG_CANDIDATES.md`. Normal continuation should read this file only when this subsystem is active.

### BUG-DUNGEON-001 — server Warp/Ranking index is unchecked
- Statik durum: **doğrulandı**
- Sınıf: server OOB / modified-client crash surface

Both `Warp` and `Ranking` access `s_vecDungeonProto[byIndex]` before validating `byIndex < size()`.
CG packet exposes uint8 index directly.

### BUG-DUNGEON-002 — client dungeon array has 255 slots but uint8 index can be 255
- Statik durum: **doğrulandı**
- Sınıf: client OOB

`m_vecDungeonInfoDataMap[255]` valid indices are 0..254.
`AddDungeon(uint8_t byIndex,...)` and many getters index it directly; 255 is representable by the network/Python boundary.

### BUG-DUNGEON-003 — CPythonDungeonInfo::Clear clears only first dungeon vector
- Statik durum: **doğrulandı**
- Sınıf: stale state / reload corruption

`m_vecDungeonInfoDataMap->clear()` is equivalent to clearing element 0 only.
Slots 1..254 retain old packet data while count/load flags reset.

### BUG-DUNGEON-004 — Warp couples level-limit count to entry-position vector
- Statik durum: **doğrulandı**
- Sınıf: server OOB / malformed-config crash

Warp loops `iPos < vecLevelLimit.size()` and indexes `vecEntryPosition[iPos]`.
No invariant check guarantees both config vectors have equal sizes.

### BUG-DUNGEON-005 — variable config item vectors copied into fixed packet arrays without cap
- Statik durum: **doğrulandı**
- Sınıf: stack/packet memory overwrite from malformed config

`SendInfo` loops full `vecRequiredItem` and `vecBossDropItem` and writes fixed `sRequiredItem[]` / `sBossDropItem[]` arrays with no size cap.

### BUG-DUNGEON-006 — bonus bounds check is off by one
- Statik durum: **doğrulandı**
- Sınıf: packet stack OOB

The loop breaks only when `iAffect > POINT_MAX_NUM`.
Index `POINT_MAX_NUM` is already outside arrays sized `[POINT_MAX_NUM]`; guard must stop before equality.

### BUG-DUNGEON-007 — LoadFile copies config tokens with unbounded strcpy
- Statik durum: **doğrulandı**
- Sınıf: config parser / stack buffer overflow

`CDungeonInfoManager::LoadFile` reads lines up to 512 bytes, but copies token 1/2/3 with raw `strcpy` into `char szValue1[QUEST_NAME_MAX_LEN]`, `szValue2[...]`, and `szValue3[...]`.

A malformed or future `dungeon_info.txt` line with a token longer than the destination buffer can overwrite the stack during server startup or `/reload dungeon`. The current checked-in config uses short tokens, so the present data snapshot does not trigger it.

### BUG-DUNGEON-008 — Python DungeonInfo getter APIs expose unchecked nested slot/type indices
- Statik durum: **doğrulandı**
- Sınıf: client OOB read / crash

Bonus getters accept `uint16_t` dungeon/type indices and directly index the 255-element dungeon storage plus fixed bonus arrays. Required-item and boss-drop getters accept byte slots and directly index `sRequiredItem[slot]` / `sBossDropItem[slot]`. The Python module adds no bounds checks.

Crafted Python/UI calls can therefore read outside both the dungeon container and nested packet arrays. This is distinct from BUG-DUNGEON-002, which covers the primary dungeon-index 255 boundary.

### BUG-DUNGEON-009 — Ranking SQL concatenates table name and LEFT JOIN without whitespace
- Statik durum: **doğrulandı**
- Sınıf: SQL correctness / ranking feature failure

`CDungeonInfoManager::Ranking` builds adjacent string literals with no whitespace between the closing `dungeon_ranking` table identifier and `LEFT JOIN`. The resulting SQL contains `dungeon_ranking\`LEFT JOIN`, which is invalid SQL. The function returns on `uiSQLErrno`, so normal ranking requests fail through this query.

### BUG-DUNGEON-010 — RETRACTED: list-button loop was misread as nested in zero-count branch
- Statik durum: **RETRACTED / FALSE POSITIVE (2026-09-29)**
- Sınıf: mapping correction

Fresh source verification shows the `for key in xrange(...)` list-button loop is aligned after the `if/else`; it is not inside the zero-count branch. With `GetCount() > 0`, controls are unlocked and the loop still executes. This ID must not be treated as an active bug.

### BUG-DUNGEON-011 — config documents numeric GLOBAL flag but parser expects literal GLOBAL
- Statik durum: **doğrulandı**
- Sınıf: config contract / cooldown source selection

The checked-in config documents third QUEST token `0 = PC`, `1 = GLOBAL`. The server parser marks a quest global only when that token is exactly the string `GLOBAL`; every other value becomes PC.

Current data contains `QUEST dragon_lair_access dragon_lair_time 1`, so it is loaded as a PC quest flag although the config format declares it GLOBAL.

### BUG-DUNGEON-012 — expired/zero cooldown arithmetic wraps negative time into huge uint32 value
- Statik durum: **doğrulandı**
- Sınıf: unsigned underflow / normal-state cooldown corruption

In `SendInfo`, the expired-path calculation assigns `(dwFlagValue + dwCooldown) - get_global_time()` directly to `uint32_t dwRemainSec`. A zero or already-expired value produces a negative mathematical result, wraps to a very large positive integer, and is then accepted by `if (dwRemainSec > 0)` as an active cooldown.

The current `dungeon_info.txt` contains QUEST-backed dungeons and no explicit `COOLDOWN` lines, so unset/expired flags can hit this path in the normal snapshot.

## Battle Pass — recovered canonical bugs
