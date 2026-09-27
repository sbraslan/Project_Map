# CURRENT — Canonical Active Checkpoint

**Active phase:** Detection / Mapping Only
**Active subsystem:** Sung Mahi Tower
**Status:** PARTIAL — ACTIVE
**Machine state:** `STATE.json`
**Canonical map:** `systems/sung_mahi_tower.md`
**Verified bug registry:** `bugs/sung_mahi_tower.md`
**Last updated:** 2026-09-27

## Hard rule
Only `sbraslan/Project_Map` is writable. All source/game repositories remain read-only.

## Just mapped
- `m_bDungeon_Difficulty` and dungeon flag `dungeonLevel` are fully independent C++ states; no automatic synchronization exists.
- `d.clear_dungeon_flags()` clears `dungeonLevel` but does not reset `m_bDungeon_Difficulty`.
- Group-spawned monsters receive `SetDungeonMultipliers()`; tower-specific `d.spawn_mob_dir_nomove()` individual spawns do not.
- `pc.mailbox_reward` calls through `ch->GetMailBox()` without a null guard; this remains deferred because the tracked tower quest caller is missing.
- Initial client command timing is now closed as non-bug: tower cover is shown during Loading-phase Warp/map setup, while quest login execution is server-gated until `PHASE_GAME`.
- Pre-game `servercommandparser.py` does not know Sung Mahi commands, but no mapped normal quest producer can send them before PHASE_GAME.
- Verified bugs remain: `BUG-SMT-001`, `BUG-SMT-002`.

## Exact next work
1. Trace generic quest kill/leave/logout/dungeon-destroy hooks used for tower room completion and cleanup.
2. Audit monthly reward event restart/month-transition/mail-write edge cases.
3. Revisit `pc.mailbox_reward` null-mailbox safety only if a callable tower producer is recovered.
4. Treat ranking row production and dual floor-state synchronization as missing-quest responsibilities unless another producer is found.
5. No production source changes.

GitHub state is canonical.
