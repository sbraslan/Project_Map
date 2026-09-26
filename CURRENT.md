# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Party Match
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/party_match.md`
**Last updated:** 2026-09-26

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Closed this turn — core Party
Party System is now **STATIC COMPLETE**.

Verified core Party bugs:
- `BUG-PARTY-001`
- `BUG-PARTY-002`
- `BUG-PARTY-003`
- `BUG-PARTY-004`
- `BUG-PARTY-005`
- `BUG-PARTY-006`

Final closure also covered:
- item/drop ownership rotation;
- EXP-centralize producer reachability;
- remaining Party packet layout;
- remaining quest helpers.

## Active Party Match checkpoint
Mapped:
- separate `CGroupMatchManager`;
- local SearchMap queue;
- search/cancel CG/GC protocol;
- validation, item checks and match completion;
- normal Party creation handoff;
- logout queue cleanup;
- client UI state;
- active common client config vs server Coordinates.

No verified Party Match bug yet.

## Exact next work
1. Close cross-core queue scope.
2. Close duplicate SEARCH/HOLD desync reachability.
3. Audit item-consumption/warp atomicity.
4. Audit raw character pointer lifetime.
5. Audit completion/failure queue cleanup.

GitHub state is canonical.
