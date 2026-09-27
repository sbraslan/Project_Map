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
- Generic completion/cleanup lifecycle is closed: quest kill/logout hooks, dungeon membership teardown, delayed destroy, private-map server-timer cancellation.
- No dedicated C++ Sung Mahi room-completion controller exists; floor progression/reward/ranking orchestration is expected from the missing quest runtime.
- Verified `BUG-SMT-003`: monthly mailbox reward uses unsafe fixed-width memcpy; title is deterministically non-NUL-terminated and shorter literals are over-read.
- Verified `BUG-SMT-004`: persistent season marker stores only `tm_mon` (0..11), so long downtime ending in the same month number in a later year skips rollover.
- DB event flags are persisted/restored correctly for normal restarts; the defect is specifically month-only season identity.
- Client floor model and shipped reward/element tables align on valid floors 1..50; latent malformed-input off-by-one guards were found but are not promoted as valid-flow bugs.
- Verified bugs: `BUG-SMT-001`, `BUG-SMT-002`, `BUG-SMT-003`, `BUG-SMT-004`.

## Exact next work
1. Inspect remaining independent monthly-reward/mailbox edge cases.
2. Review `smhtower_*` monster/group and map data integration for static mismatches.
3. Revisit `pc.mailbox_reward` null-mailbox safety only if a callable tower producer is recovered.
4. Decide whether the subsystem is ready for STATIC COMPLETE with missing runtime behavior explicitly represented by BUG-SMT-001.
5. No production source changes.

GitHub state is canonical.
