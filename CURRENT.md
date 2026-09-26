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
New unmapped subsystem opened: **Ranking System**.

Mapped:
- server ranking manager and BattleField DB lifecycle;
- BattleField score persistence / weekly rollover;
- P2P ranking-reload receive path;
- GC dynamic ranking packet;
- client C++ ranking cache;
- Python ranking module;
- active BattleField ranking UI;
- generic ranking-board integration.

Verified bugs:
- `BUG-RANK-001` — empty ranking vector indexed with `&vec[0]`.
- `BUG-RANK-002` — current-player ranking API is a hard-coded empty stub.
- `BUG-RANK-003` — weekly winner table can retain stale prior-week positions.
- `BUG-RANK-004` — BattleField close reloads cache before final player scores are persisted.

## Not promoted yet
- PARTY ranking UI references Python APIs that are not exported, but no live PARTY opener has been located.
- Generic SOLO categories 2..7 lack UI name entries, but no live opener has been located.
- Ranker winner effects are not visibly refreshed for already-online players on ranking reload; exact lifecycle still needs closure.

## Exact next work
1. Locate the exact outbound `TPacketGGLoadRanking` sender/broadcast path.
2. Close ranker-effect refresh/removal lifecycle.
3. Determine whether any active caller opens generic PARTY ranking.
4. Audit dynamic GC ranking packet malformed-size handling.
5. Audit remaining DB/result null boundaries and decide Ranking STATIC COMPLETE.

## Startup
For the next normal "ilerleyelim":
1. read `STATE.json`;
2. read `CURRENT.md`;
3. read `systems/ranking.md`;
4. read `bugs/ranking.md` only when validating/adding a bug;
5. inspect exact source symbols read-only;
6. write findings only to `Project_Map`.

GitHub state is canonical. Do not reconstruct from old chats.
