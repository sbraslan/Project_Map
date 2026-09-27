# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** World Lottery System
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/world_lottery.md`
**Last updated:** 2026-09-27

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Just closed
Battle Field System -> **STATIC COMPLETE**.

Verified Battle Field bugs:
- `BUG-BFIELD-001..011`

The Battle Field system file was closed in a later commit than the stale STATE/CURRENT cursor. This checkpoint reconciles that drift.

## Active World Lottery direction
Initial roots to map:
- server lottery character/manager code;
- lottery packet definitions and packet-info registration;
- DB persistence/tables;
- client network receive/send handlers;
- Python module/UI;
- ranking/jackpot lifecycle and draw scheduling.

## Verified World Lottery bugs
- `BUG-WLOT-001` — arbitrary server ticket slots bypass the intended 3-ticket limit.
- `BUG-WLOT-002` — duplicate selected numbers can turn one real match into a 4/4 jackpot result.
- `BUG-WLOT-003` — long long lottery values narrow through `PointChange(int)`.
- `BUG-WLOT-004` — lottery wallet is deducted before gold-cap rejection.
- `BUG-WLOT-005` — out-of-range stored ticket numbers can break official client lottery refresh.
- `BUG-WLOT-006` — negative withdrawal amounts can drive gold negative while increasing the lottery wallet.

## Exact next work
1. Finish ticket delete/claim state transitions and SQL-error/null handling.
2. Audit COUNT(*)-based draw-id assumptions and row continuity.
3. Audit multiple-jackpot accounting and next-jackpot math.
4. Audit log lotto_id/ticket_id mapping.
5. Audit claim/withdrawal persistence and save ordering.
6. Audit ranking empire lookup/result safety.

Client receive/cache/reset behavior is now mapped. Ranking cache is append-only/stale but currently has no shipped UI caller, so it remains a dormant note rather than a verified user-facing bug.

Record only findings in Project_Map.

GitHub state is canonical.
