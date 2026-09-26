# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Party System
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/party.md`
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

## Closed this turn — Ranking
Ranking System is now **STATIC COMPLETE**.

Verified Ranking bugs:
- `BUG-RANK-001`
- `BUG-RANK-002`
- `BUG-RANK-003`
- `BUG-RANK-004`
- `BUG-RANK-005`
- `BUG-RANK-007`

Retracted/reserved:
- `BUG-RANK-006` — false positive.

## Active Party checkpoint
Initial Party roots are mapped:
- game `party.h/.cpp`;
- DB `ClientManagerParty.cpp`;
- game request handlers in `input_main.cpp`;
- client network/player layer;
- `root/uiparty.py` / interface / game bridge.

Known architecture:
- game creates/changes parties and reports to DB;
- DB tracks party state per channel and forwards party changes to peers;
- game exposes invite/answer/state/remove/skill/parameter handlers;
- client UI contains role, member, heal/warp, leave/disband and EXP distribution actions.

## Exact next work
1. Map Party Create -> DB -> peer replication.
2. Map Join / Leave / Delete lifecycle.
3. Map CG handlers and validation.
4. Map GC client cache/UI updates.
5. Start bug-candidate audit.

GitHub state is canonical. Do not reconstruct from old chats.
