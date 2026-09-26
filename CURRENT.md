# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Battle Field System
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/battle_field.md`
**Last updated:** 2026-09-27

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Just closed
Dungeon Core -> **STATIC COMPLETE**.

Verified Dungeon Core bugs:
- `BUG-DUNGEON-001`
- `BUG-DUNGEON-002`
- `BUG-DUNGEON-003`
- `BUG-DUNGEON-004`

Regen, bulk purge iteration, manager event identity, private-map teardown and remaining unpromoted lifecycle candidates were closed statically.

## Active Battle Field direction
Initial roots:
- `game/src/battle_field.cpp/.h`
- Battle Field commands/P2P state propagation
- Battle Field packets
- `root/uibattlefield.py`
- client player/network bindings
- Ranking integration boundaries

## Verified Battle Field bugs
- `BUG-BFIELD-001` — player `exit_battle_field` works outside Battle Field wherever CanWarp permits.
- `BUG-BFIELD-002` — `exit_battle_field_on_dead 1` directly executes Battle Field exit without map/death/CanWarp validation.
- `BUG-BFIELD-003` — repeat-kill cooldown timestamp is not renewed after first expiry.

## Exact next work
1. Finish schedule/open-close resolver correctness.
2. Audit event-mode/P2P state across cores.
3. Trace Battle Field party removal into existing Party invariants.
4. Audit score cash-out/ranking persistence and disconnect behavior.
5. Audit daily Battle shop-point reset.

GitHub state is canonical.
