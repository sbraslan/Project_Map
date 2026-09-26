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

No source/game files may be edited.

## Party checkpoint
Mapped:
- create/join/remove/delete + DB channel replication;
- client CG/GC bridge and normal cache cleanup;
- invite/accept revalidation;
- leader/member authority for role/remove/skill/parameter operations;
- reconnect/offline ADD/LINK behavior;
- role-counter integrity;
- minimap party-position dynamic packet parsing.

Verified bugs:
- `BUG-PARTY-001` — leader `Quit()` use-after-free after self-deleting `P2PQuit()`.
- `BUG-PARTY-002` — Party Heal is hard-disabled server-side.
- `BUG-PARTY-003` — mismatched role-off packet corrupts role counters.
- `BUG-PARTY-004` — malformed dynamic party-position size can underflow/desynchronize parsing.

## Audited non-bugs
- invite accept requires a real pending server event and revalidates mutable conditions;
- warp verifies target PID belongs to the party;
- EXP mode is server bounds-checked;
- normal GC remove clears C++ party cache through Python/player bridge;
- reconnect/offline PID handling does not duplicate party boards on the mapped normal path.

## Exact next work
1. Map near-member/bonus/EXP lifecycle.
2. Audit map/channel position helpers across cores/channels.
3. Audit quest party APIs against party lifetime/delete.
4. Separate/audit Party Match.
5. Continue packet/state boundaries and decide Party STATIC COMPLETE.

GitHub state is canonical.
