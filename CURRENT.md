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

## Exact next work
1. Audit entry/exit command authorization and channel/map restrictions.
2. Audit kill/death/score accounting and persistence.
3. Audit schedule day/time calculations.
4. Audit cooldown/reconnect behavior.
5. Audit P2P state synchronization.
6. Record only Battle Field-specific bugs; Ranking bugs stay in Ranking registry.

GitHub state is canonical.
