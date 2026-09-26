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

## Party checkpoint
Mapped:
- Create / Join / Remove / Delete lifecycle;
- GD -> DB channel map -> DG peer replication;
- game-side DB receive handlers;
- invite/accept authority and mutable-condition revalidation;
- role/remove/skill/EXP parameter authority;
- GC party packet bridge and normal client cache removal;
- party destructor/member-map cleanup.

Verified bugs:
- `BUG-PARTY-001` — leader `Quit()` returns from a self-deleting `P2PQuit()` and uses the freed `CParty`.
- `BUG-PARTY-002` — Party Heal UI/server integration is hard-disabled; heal cannot execute.

## Exact next work
1. Audit reconnect/offline member ADD/LINK/UNLINK and duplicate PID cache behavior.
2. Audit `SetRole` internal bounds across DB/P2P paths.
3. Map near-member/bonus/EXP update lifecycle.
4. Separate/audit Party Match.
5. Continue packet/state consistency checks.

GitHub state is canonical. Do not reconstruct from old chats.
