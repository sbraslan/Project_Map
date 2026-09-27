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
World Lottery System -> **STATIC COMPLETE** with `BUG-WLOT-001..014`.

## World Boss progress
Mapped:
- event/config constants and scheduler;
- P2P state packet/fanout;
- boss death/ranking path;
- reward command/state;
- Python World Boss state/ranking UI;
- timeout/event-disable/login lifecycle;
- multi-core/channel ownership;
- ranking global-cache lifecycle.

Verified:
- `BUG-WB-001..014`.

## Exact next work
1. Finish tier-assignment provenance with a complete caller scan if repository search becomes available.
2. Finish reward-state reset provenance (`SetWBRewards(false)`) beyond mapped World Boss roots.
3. Finish tier/reward provenance beyond the known World Boss roots.
4. Audit remaining client open/close/state-reset lifecycle.
5. Audit World Boss configuration/data roots outside C++ (event/quest/config if present).
6. Decide World Boss STATIC COMPLETE only after provenance is closed.

Record only findings in Project_Map. Source repositories stay immutable.

GitHub state is canonical.
