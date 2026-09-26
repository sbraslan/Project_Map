# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Ranking System
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/ranking.md`
**Last updated:** 2026-09-26

## Hard rule
Only `sbraslan/Project_Map` is writable.

Strictly read-only:
- `Project_ClientSrc`
- `Project_ServerSRC`
- `Project_Binary`
- `Project_Game`
- `Project_DumpProto`

No C++, Python, quest, config, game-data, source or binary file may be edited, committed or pushed.

## Ranking checkpoint
Closed this turn:
- live UI opener mapped: minimap BattleButton -> `/open_battle_ui` -> server command -> `OpenBattleUI` -> ranking GC packet -> BattleField UI;
- outbound P2P reload sender found in `cmd_general.cpp::LoadRanking(uint8_t)`;
- sender broadcasts `TPacketGGLoadRanking` to P2P peers and reloads local cache;
- former `BUG-RANK-006` retracted: `cmd.h` declaration is visible through `battle_field.cpp -> char.h -> horse_rider.h -> cmd.h`;
- online ranker-effect lifecycle closed and promoted to `BUG-RANK-007`.

Verified bugs now:
- `BUG-RANK-001`
- `BUG-RANK-002`
- `BUG-RANK-003`
- `BUG-RANK-004`
- `BUG-RANK-005`
- `BUG-RANK-007`

Retracted/reserved:
- `BUG-RANK-006` — false positive; do not reuse ID.

## Remaining open mapping
- Generic PARTY board calls APIs not exported by `PythonRankingModule.cpp`; live opener still not found.
- Generic SOLO categories 2..7 lack UI name entries; live opener still not found.
- Remaining ranking SQL/result null boundaries need final audit.
- Additional direct ranker-flag cleanup paths may affect BUG-RANK-007 lifetime, but missing refresh on ranking reload is already verified.

## Exact next work
1. Determine whether any active caller opens generic PARTY ranking.
2. Determine whether any active caller opens generic SOLO categories 2..7.
3. Audit remaining ranking SQL/result null boundaries.
4. Check additional direct ranker-flag cleanup paths for BUG-RANK-007 lifetime.
5. Decide Ranking STATIC COMPLETE.

GitHub state is canonical. Do not reconstruct from old chats.
