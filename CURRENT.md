# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Sung Mahi Tower
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/sung_mahi_tower.md`
**Last updated:** 2026-09-27

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Just closed
World Boss System -> **STATIC COMPLETE** with `BUG-WB-001..016`.

## Sung Mahi Tower progress
- Feature flag confirmed enabled.
- Client command/UI roots found in `Project_Binary/root/game.py`, `uisungmahi.py`, and two UIScript files.
- Server behavior is embedded in generic Yohara/character/battle code rather than a dedicated tower source file.
- Initial combat/map-data roots are mapped; server command producers remain next.

## Exact next work
1. Find server producers for the Sung Mahi client command names.
2. Trace `uisungmahi.py` room/tower/reward lifecycle.
3. Trace SungMa map-attribute loading and combat enforcement.
4. Map Conqueror/tower persistence and reward boundaries.
5. Record only verified bugs after end-to-end flow closure.

GitHub state is canonical.
