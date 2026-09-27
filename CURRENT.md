# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** World Boss System
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/world_boss.md`
**Last updated:** 2026-09-27

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Just closed
World Lottery System -> **STATIC COMPLETE**.

Verified World Lottery bugs:
- `BUG-WLOT-001..014`

## Active World Boss direction
Initial client roots:
- `root/uiworldboss.py`
- `root/uiworldbossranking.py`
- `root/uiscript/worldbosswindow.py`
- `root/uiscript/worldbossrankingwindow.py`

Server-side World Boss logic is not stored in a dedicated filename. Trace exact feature flags, packets and callbacks from generic server/client files.

## Exact next work
1. Trace `ENABLE_WORLD_BOSS` and World Boss packet definitions/handlers.
2. Map spawn/state/reward/ranking server ownership.
3. Map client receive callbacks and Python ranking/cache lifecycle.
4. Record verified bugs only in Project_Map.

GitHub state is canonical.
