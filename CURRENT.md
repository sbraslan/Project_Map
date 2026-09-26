# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Party Match
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/party_match.md`
**Last updated:** 2026-09-26

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Party Match checkpoint
Mapped:
- local SearchMap queue and search/cancel protocol;
- validation and match completion;
- common client config vs server Coordinates;
- logout queue cleanup;
- multi-core deployment topology;
- P2P/DB replication absence;
- WarpSet failure contract.

Verified bugs:
- `BUG-PMATCH-001` — same-channel search pool is fragmented per game core.

## Key evidence
CH1 deploys core1..core5 as separate game processes, all with `CHANNEL: 1`.

Party Match has no GG/P2P or GD/DG state replication. Each core therefore matches only players currently attached to that process.

## Audited/non-promoted
- normal disconnect removes raw LPCHARACTER queue entries;
- active common Party Match config matches server map/level/item requirements;
- country/ae Party Match file is not used by the active loader path;
- WarpSet result is ignored after item consumption, but no active configured failure path is yet verified.

## Exact next work
1. Close duplicate SEARCH/HOLD desync.
2. Audit map/core changes while queued.
3. Audit success-vs-WarpSet failure behavior.
4. Audit inventory/item-removal restrictions.
5. Audit UI/minimap state for all result codes.

GitHub state is canonical.
