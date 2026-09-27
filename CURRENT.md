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
- event/config constants;
- spawn/kill scheduler;
- P2P state packet/fanout;
- boss death hook;
- server damage-ranking emission;
- normal-player reward command;
- Python World Boss state/ranking callbacks;
- persistent interface windows and UIScript controls.

Verified:
- `BUG-WB-001..012`.

## Exact next work
1. Finish tier-assignment provenance / determine whether reward tier is ever assigned.
2. Audit login/reconnect state synchronization.
3. Audit tier/reward reset and persistence across boss cycles.
4. Audit multi-core/channel boss ownership.
5. Audit ranking cache reset/pagination after upstream failures.

Damage-ranking ownership iteration and timeout/event-disable lifecycle are now statically mapped.

Record only findings in Project_Map. Source repositories stay immutable.

GitHub state is canonical.
