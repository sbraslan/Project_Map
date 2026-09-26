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
- `BUG-PMATCH-001` — same-channel matchmaking pool fragmented per game core.
- `BUG-PMATCH-002` — required exchange-listed item can be destroyed while CExchange retains a raw pointer.
- `BUG-PMATCH-003` — queued FAIL_NO_ITEM resets main state but leaves minimap Party Match icon visible.

## Client state closure
SEARCH/CANCEL Python argument shapes are odd but intentionally matched by `uiPartyMatch.PartyMatchResult`.

Duplicate SEARCH/HOLD can desync client/server state, but stock UI sends CANCEL after the first search, so HOLD remains a robustness finding rather than a separate active bug.

## Exact next work
1. Audit remaining conflicting item windows.
2. Audit success notification vs WarpSet failure.
3. Audit queued map/core transitions.
4. Audit Party Match off/disable enforcement.
5. Decide Party Match STATIC COMPLETE.

GitHub state is canonical.
