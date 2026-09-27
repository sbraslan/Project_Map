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
- `BUG-WLOT-005` — out-of-range ticket numbers can break official client ticket refresh.
- `BUG-WLOT-006` — negative withdrawal can drive gold negative while increasing lottery wallet.
- `BUG-WLOT-007` — COUNT(*) is incorrectly used as current/future draw identity.
- `BUG-WLOT-008` — result logs store the previous draw id.
- `BUG-WLOT-009` — multiple jackpot winners each receive a full jackpot.
- `BUG-WLOT-010` — recurring draw schedule hardcodes 30s instead of configured 2 minutes.
- `BUG-WLOT-011` — dormant ranking path sends ticket id as lottoID.
- `BUG-WLOT-012` — dormant ranking path can null-dereference a missing empire row.

## Exact next work
1. Finish ticket delete/claim SQL-error/null handling and mutation ordering.
2. Audit async result-update ordering versus next-draw insertion.
3. Audit claim/withdrawal persistence and save ordering.
4. Check server->client lottery point serialization widths.
5. Reconcile World Lottery and determine STATIC COMPLETE readiness.

Client cache, draw-id, jackpot, log mapping, scheduling, and ranking safety are now statically mapped.

Record only findings in Project_Map.

GitHub state is canonical.
