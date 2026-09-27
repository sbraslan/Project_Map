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

## Exact next work
1. Locate all World Lottery server/client roots and packet headers.
2. Map ticket purchase -> persistence -> draw -> reward flow.
3. Audit client-controlled number/count/price inputs.
4. Audit jackpot/money caps and winner payout ordering.
5. Audit ranking packet sizes, array/count boundaries and stale-state resets.
6. Record only findings in Project_Map.

GitHub state is canonical.
