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
Verified bugs:
- `BUG-PMATCH-001` — same-channel matchmaking pool is fragmented per game core.
- `BUG-PMATCH-002` — Party Match can consume/destroy a required item that is still referenced by an active exchange, leaving a dangling exchange item pointer.

## PMATCH-002 proof chain
`CExchange::AddItem`
-> raw `m_apItems[]` + `SetExchanging(true)`
-> Party Match `CountSpecifyItem` still counts it
-> `RemoveSpecifyItem` still removes it
-> exact stack `SetCount(0)`
-> `ITEM_MANAGER::DestroyItem`
-> `M2_DELETE(item)`
-> exchange pointer remains
-> later `CExchange::Cancel` dereferences it.

`CHARACTER::Destroy` automatically cancels an active exchange, so cross-core Party Match warp gives a normal follow-up lifecycle path.

## Audited/non-promoted
- normal logout removes SearchMap raw character entry;
- active common config matches server requirements;
- ignored WarpSet return is still a non-atomic weakness but lacks a verified configured normal failure path.

## Exact next work
1. Close duplicate SEARCH/HOLD desync.
2. Audit safebox/refine/shop/change-look conflicting item states.
3. Audit success-vs-WarpSet ordering.
4. Audit client UI/minimap result state.
5. Continue Party Match closure.

GitHub state is canonical.
