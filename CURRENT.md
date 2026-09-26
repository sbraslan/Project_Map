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
Mapped this turn:
- targeted P2P sender audit: `TPacketGGLoadRanking` receiver/struct found, outbound sender still not found;
- ranker effects are direct affect-flag bits, not normal `CAffect` entries in the mapped path;
- dynamic BattleField ranking packet boundary handling audited;
- BattleField ranking reload call sites audited against the active source configuration.

New verified bugs:
- `BUG-RANK-005` — malformed dynamic ranking size can underflow/cross the declared packet boundary.
- `BUG-RANK-006` — active BattleField source calls unresolved `LoadRanking(RK_CATEGORY_BF)`.

Previously verified:
- `BUG-RANK-001`
- `BUG-RANK-002`
- `BUG-RANK-003`
- `BUG-RANK-004`

## Not promoted yet
- No outbound `TPacketGGLoadRanking` sender has been located; cross-channel stale-cache impact needs OpenBattleUI reachability.
- Ranker winner bits are not cleared in the mapped lifecycle; repo-wide direct-reset search still needs closure.
- Generic PARTY ranking APIs are absent from the Python module, but no live PARTY opener has been located.
- Generic SOLO categories 2..7 still lack UI name entries, but no live opener has been located.

## Exact next work
1. Map all live callers of `CBattleField::OpenBattleUI` and test the missing-P2P-sender stale-cache hypothesis statically.
2. Finish repo-wide direct-reset search for `AFF_BATTLE_RANKER_1..3`.
3. Determine whether any active caller opens generic PARTY ranking.
4. Audit remaining ranking SQL/result null boundaries.
5. Reconcile BUG-RANK-006 with actual build entry points/configuration, then decide Ranking STATIC COMPLETE.

## Startup
For the next normal "ilerleyelim":
1. read `STATE.json`;
2. read `CURRENT.md`;
3. read `systems/ranking.md`;
4. read `bugs/ranking.md` only when validating/adding a bug;
5. inspect exact source symbols read-only;
6. write findings only to `Project_Map`.

GitHub state is canonical. Do not reconstruct from old chats.
