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

## Battle Pass — recovered canonical bugs
